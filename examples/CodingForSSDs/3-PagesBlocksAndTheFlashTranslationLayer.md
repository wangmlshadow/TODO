# Coding for SSDs – Part 3: Pages, Blocks, and the Flash Translation Layer

This is Part 3 over 6 of “Coding for SSDs”, covering Sections 3 and 4. For other parts and sections, you can refer to the [Table to Contents](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/). This is a series of articles that I wrote to share what I learned while documenting myself on SSDs, and on how to make code perform well on SSDs. If you’re in a rush, you can also go directly to [Part 6](http://codecapsule.com/2014/02/12/coding-for-ssds-part-6-a-summary-what-every-programmer-should-know-about-solid-state-drives/), which is summarizing the content from all the other parts.

In this part, I am explaining how writes are handled at the page and block level, and I talk about the fundamental concepts of write amplification and wear leveling. Moreover, I describe what is a Flash Translation Layer (FTL), and I cover its two main purposes, logical block mapping and garbage collection. More particularly, I explain how write operations work in the context of a hybrid log-block mapping.

 **Translations** : This article was translated to [Simplified Chinese](http://blog.xiongduo.cn/IT-Trans/oding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer.html) by Xiong Duo and to [Korean](http://tech.kakao.com/2016/07/15/coding-for-ssd-part-3/) by Matt Lee (이 성욱).

![ssd-presentation-03](https://i0.wp.com/codecapsule.com/wp-content/uploads/2014/02/ssd-presentation-03.jpg?resize=720%2C505)

## 3. Basic operations

### 3.1 Read, write, erase

Due to the organization of NAND-flash cells, it is not possible to read or write single cells individually. Memory is grouped and is accessed with very specific properties. Knowing those properties is crucial for optimizing data structures for solid-state drives and for understanding their behavior. I am describing below the basic properties of SSDs regarding the read, write and erase operations.

#### Reads are aligned on page size

It is not possible to read less than one page at once. One can of course only request just one byte from the operating system, but a full page will be retrieved in the SSD, forcing a lot more data to be read than necessary.

#### Writes are aligned on page size

When writing to an SSD, writes happen by increments of the page size. So even if a write operation affects only one byte, a whole page will be written anyway. Writing more data than necessary is known as write amplification, a concept that is covered in Section 3.3. In addition, writing data to a page is sometimes referred to as “to program” a page, therefore the terms “write” and “program” are used interchangeably in most publications and articles related to SSDs.

#### Pages cannot be overwritten

A NAND-flash page can be written to only if it is in the “free” state. When data is changed, the content of the page is copied into an internal register, the data is updated, and the new version is stored in a “free” page, an operation called “read-modify-write”. The data is not updated in-place, as the “free” page is a different page than the page that originally contained the data. Once the data is persisted to the drive, the original page is marked as being “stale”, and will remain as such until it is erased.

#### Erases are aligned on block size

Pages cannot be overwritten, and once they become stale, the only way to make them free again is to erase them. However, it is not possible to erase individual pages, and it is only possible to erase whole blocks at once. From a user perspective, only read and write commands can be emitted when data is accessed. The erase command is triggered automatically by the garbage collection process in the SSD controller when it needs to reclaim stale pages to make free space.

### 3.2 Example of a write

Let’s illustrate the concepts from Section 3.1. Figure 4 below shows an example of data being written to an SSD. Only two blocks are shown, and those blocks contain only four pages each. This is obviously a simplified representation of a NAND-flash package, created for the sake of the reduced examples I am presenting here. At each step in the figure, bullet points on the right of the schematics explain what is happening.

![ssd-writing-data](https://i0.wp.com/codecapsule.com/wp-content/uploads/2014/02/ssd-writing-data.jpg?resize=720%2C761)

Figure 4: Writing data to a solid-state drive

### 3.3 Write amplification

Because writes are aligned on the page size, any write operation that is not both aligned on the page size and a multiple of the page size will require more data to be written than necessary, a concept called **write amplification** [[13]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). Writing one byte will end up writing a page, which can amount up to 16 KB for some models of SSD and be extremely inefficient.

But this is not the only problem. In addition to writing more data than necessary, those writes also trigger more internal operations than necessary. Indeed, writing data in an unaligned way causes the pages to be read into cache before being modified and written back to the drive, which is slower than directly writing pages to the disk. This operations is known as  **read-modify-write** , and should be avoided whenever possible [[2, 5]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

#### Never write less than a page

Avoid writing chunks of data that are below the size of a NAND-flash page to minimize write amplification and prevent read-modify-write operations. The largest size for a page at the moment is 16 KB, therefore it is the value that should be used by default. This size depends on the SSD models and you may need to increase it in the future as SSDs improve.

#### Align writes

Align writes on the page size, and write chunks of data that are multiple of the page size.

#### Buffer small writes

To maximize throughput, whenever possible keep small writes into a buffer in RAM and when the buffer is full, perform a single large write to batch all the small writes.

### 3.4 Wear leveling

As discussed in Section 1.1, NAND-flash cells have a limited lifespan due to their limited number of P/E cycles. Let’s imagine that we had an hypothetical SSD in which data was always read and written from the same exact block. This block would quickly exceed its P/E cycle limit, wear off, and the SSD controller would mark it as being unusable. The overall capacity of the disk would then decrease. Imagine buying a 500 GB drive and being left with at 250 GB a couple of years later, that would be outrageous!

For that reason, one of the main goals of an SSD controller is to implement  **wear leveling** , which distributes P/E cycles as evenly as possible among the blocks. Ideally, all blocks would reach their P/E cycle limits and wear off at the same time [[12, 14]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

In order to achieve the best overall wear leveling, the SSD controller will need to choose blocks judiciously when writing, and may have to move around some blocks, a process which in itself incurs an increase of the write amplification. Therefore, block management is a trade-off between maximizing wear leveling and minimizing write amplification.

Manufacturers have come up with various functionalities to achieve wear leveling, such as garbage collection, which is covered in the next section.

#### Wear leveling

Because NAND-flash cells are wearing off, one of the main goals of the FTL is to distribute the work among cells as evenly as possible so that blocks will reach their P/E cycle limit and wear off at the same time.

## 4. Flash Translation Layer (FTL)

### 4.1 On the necessity of having an FTL

The main factor that made adoption of SSDs so easy is that they use the same host interfaces as HDDs. Although presenting an array of Logical Block Addresses (LBA) makes sense for HDDs as their sectors can be overwritten, it is not fully suited to the way flash memory works. For this reason, an additional component is required to hide the inner characteristics of NAND flash memory and expose only an array of LBAs to the host. This component is called the *Flash Translation Layer* (FTL), and resides in the SSD controller. The FTL is critical and has two main purposes: logical block mapping and garbage collection.

### 4.2 Logical block mapping

The logical block mapping translates logical block addresses (LBAs) from the host space into physical block addresses (PBAs) in the physical NAND-flash memory space. This mapping takes the form of a table, which for any LBA gives the corresponding PBA. This mapping table is stored in the RAM of the SSD for speed of access, and is persisted in flash memory in case of power failure. When the SSD powers up, the table is read from the persisted version and reconstructed into the RAM of the SSD [[1, 5]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

The naive approach is to use a **page-level mapping** to map any logical page from the host to a physical page. This mapping policy offers a lot of flexibility, but the major drawback is that the mapping table requires a lot of RAM, which can significantly increase the manufacturing costs. A solution to that would be to map blocks instead of pages, using a  **block-level mapping** . Let’s assume that an SSD drive has 256 pages per block. This means that block-level mapping requires 256 times less memory than page-level mapping, which is a huge improvement for space utilization. However, the mapping still needs to be persisted on disk in case of power failure, and in case of workloads with a lot of small updates, full blocks of flash memory will be written whereas pages would have been enough. This increases the write amplification and makes block-level mapping widely inefficient [[1, 2]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

The tradeoff between page-level mapping and block-level mapping is the one of performance versus space. Some researcher have tried to get the best of both worlds, giving birth to the so-called “ *hybrid* ” approaches [[10]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). The most common is the  **log-block mapping** , which uses an approach similar to log-structured file systems. Incoming write operations are written sequentially to log blocks. When a log block is full, it is merged with the data block associated to the same logical block number (LBN) into a free block. Only a few log blocks need to be maintained, which allows to maintain them with a page granularity. Data blocks on the contrary are maintained with a block granularity [[9, 10]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

Figure 5 below shows a simplified representation of a hybrid log-block FTL, in which each block only has four pages. Four write operations are handled by the FTL, all having the size of a full page. The logical page numbers of 5 and 9 both resolve to LBN=1, which is associated to the physical block #1000. Initially, all the physical page offsets are null at the entry where LBN=1 is in the  *log-block mapping table* , and the log block #1000 is entirely empty as well. The first write, b’ at LPN=5, is resolving to LBN=1 by the log-block mapping table, which is associated to PBN=1000 (log block #1000). The page b’ is therefore written at the physical offset 0 in block #1000. The metadata for the mapping now needs to be updated, and for this, the physical offset associated to the logical offset of 1 (arbitrary value for this example) is updated from null to 0.

The write operations go on and the mapping metadata is updated accordingly. When the log block #1000 is entirely filled, it is merged with the data block associated to the same logical block, which is block #3000 in this case. This information can be retrieved from the  *data-block mapping table* , which maps logical block numbers to physical block numbers. The data resulting from the merge operation is written to a free block, #9000 in this case. When this is done, both blocks #1000 and #3000 can be erased and become free blocks, and block #9000 becomes a data block. The metadata for LBN=1 in the data-block mapping table is then updated from the initial data block #3000 to the new data block #9000.

An important thing to notice here is that the four write operations were concentrated on only two LPNs. The log-block approach enabled to hide the b’ and d’ operations during the merge, and directly use the more up-to-date b” and d” versions, allowing to achieve better write amplification.

Finally, if a read command is requesting a page that was recently updated and for which the merge step on the blocks has not occurred yet, then the page will be in a log block. Otherwise, the page will be found a data block. This is why read requests need to check both the *log-block mapping table* and the  *data-block mapping table* , as shown in Figure 5.

![ssd-hybrid-ftl](https://i0.wp.com/codecapsule.com/wp-content/uploads/2014/02/ssd-hybrid-ftl.jpg?resize=720%2C1009)

Figure 5: Hybrid log-block FTL

The log-block FTL allows for optimizations, the most notable being the  **switch-merge** , sometimes referred to as “swap-merge”. Let’s imagine that all the addresses in a logical block were written at once. This would mean that all the new data for those addresses would be written to the same log block. Since this log block contains the data for a whole logical block, it would be useless to merge this log block with a data block into a free block, because the resulting free block would contain exactly the data as the log block. It would be faster to only update the metadata in the data block mapping table, and switch the the data block in the data block mapping table for the log block, this is a switch-merge.

The log-block mapping scheme has been the topic of many papers, which has lead to a series of improvements, such as FAST (Fully Associative Sector Translation), superblock mapping, and flexible group mapping [[10]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). There are also other mapping schemes, such as the Mitsubishi algorithm, and SSR [[9]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). Two great starting points to learn more about the FTL and mapping schemes are the two following papers:

* “ *A Survey of Flash Translation Layer* “, Chung et al., 2009 [[9]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref)
* “ *A Reconfigurable FTL (Flash Translation Layer) Architecture for NAND Flash-Based Applications* “, Park et al., 2008 [[10]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref)

#### Flash Translation Layer

The Flash Translation Layer (FTL) is a component of the SSD controller which maps Logical Block Addresses (LBA) from the host to Physical Block Addresses (PBA) on the drive. Most recent drives implement an approach called “hybrid log-block mapping” or one of its derivatives, which works in a way that is similar to log-structured file systems. This allows random writes to be handled like sequential writes.

### 4.3 Notes on the state of the industry

As of February 2, 2014, there are 70 manufacturers of SSDs listed on Wikipedia [[64]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref), and interestingly, there are only 11 manufacturers of controllers [[65]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). Out of the 11 controller manufacturers, only four are “captive”, i.e. they use their controllers only for their own products (this is the case of Intel and Samsung), and the remaining seven are “independent”, i.e. they sell their controllers to other drive manufacturers. What those numbers are saying is that seven companies are providing controllers for 90% of the solid-state drive market.

I have no data on which controller manufacturers is selling to which drive manufacturer from these 90%, but following the Pareto principle, I would bet that only two or three controller manufacturers are sharing most of the cake. The direct consequence is that SSDs from non-captive drive manufacturers are extremely likely to behave similarly, since they are essentially running the same controllers, or at least controllers using the same general design and underlying ideas.

Mapping schemes, which are part of the controller, are critical components of SSDs because they will often entirely define the performance a drive. This explains why, in an industry with so much competition, SSD controller manufacturers do not share the details of their FTL implementations. Therefore, even though there is a lot of publicly available research regarding FTL algorithm, it is always unclear how much of that research is being used by controller manufacturers, and what are the exact implementations for specific brands and models.

The authors of [[3]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref) claim that by analyzing workloads, they can reverse engineer the mapping policies of a drive. I would argue that unless the binary code itself is being reversed engineered from the chip, there is no way to be completely sure what the mapping policy is really doing inside a specific drive. And it is even harder to predict how the mapping would behave under a specific workload.

There is a great deal of different mapping policies and it would take a considerable amount of time to reverse engineer all of the firmwares available in the wild. Then, even after getting the source code of all possible mapping policies, what would be the benefit? The system requirements of new projects are often to produce good overall results, using generic and inter-changeable commodity hardware. Therefore, it would be worthless to optimize for only one mapping policy, because the solution would be likely to perform poorly on all other mapping policies. The only reason why one would want to optimize for only one type of policy is when developing for an embedded system that is guaranteed to have consistent hardware.

For the reasons exposed above, I would argue that knowing the exact mapping policy of an SSD does not matter. The only important thing to know is that mapping schemes are the translation layer between LBAs and PBAs, and that it is very likely that the approach being used is the hybrid log-block or one of its derivatives. Consequently, writing chunks of data of at least the size of the NAND-flash block is more efficient, because for the FTL, it minimizes the overhead of updating the mapping and its metadata.

### 4.4 Garbage collection

As explained in Sections 4.1 and 4.2, pages cannot be overwritten. If the data in a page has to be updated, the new version is written to a *free* page, and the page containing the previous version is marked as  *stale* . When blocks contain stale pages, they need to be erased before they can be written to.

#### Garbage collection

The garbage collection process in the SSD controller ensures that “stale” pages are erased and restored into a “free” state so that the incoming write commands can be processed.

Because of the high latency required by the erase command compared to the write command — which are respectively 1500-3500 μs and 250-1500 μs as described in Section 1 — this extra erase step incurs a delay which makes the writes slower. Therefore, some controllers implement a  *background garbage collection process* , also called  *idle garbage collection* , which takes advantage of idle time and runs regularly in the background to reclaim stale pages and ensure that future foreground operations will have enough free pages available to achieve the highest performance [[1]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). Other implementations use a *parallel garbage collection* approach, which performs garbage collection operations in parallel with write operations from the host [[13]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

It is not uncommon to encounter workloads in which the writes are so heavy that the garbage collection needs to be run on-the-fly, at the same time as commands from the host. In that case, the garbage collection process supposed to run in background could be interfering with the foreground commands [[1]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref). The TRIM command and over-provisioning are two great ways to reduce this effect, and are covered in more details in Sections 6.1 and 6.2.

#### Background operations can affect foreground operations

Background operations such as garbage collection can impact negatively on foreground operations from the host, especially in the case of a sustained workload of small random writes.

A less important reason for blocks to be moved is the  *read disturb* . Reading can change the state of nearby cells, thus blocks need to be moved around after a certain number of reads have been reached [[14]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

The rate at which data is changing is an important factor. Some data changes rarely, and is called *cold* or *static* data, while some other data is updated frequently, which is called *hot* or *dynamic* data. If a page stores partly cold and partly hot data, then the cold data will be copied along with the hot data during garbage collection for wear leveling, increasing write amplification due to the presence of cold data. This can be avoided by splitting cold data from hot data, simply by storing them in separate pages. The drawback is then that the pages containing the cold data are less frequently erased, and therefore the blocks storing cold and hot data have to be swapped regularly to ensure wear leveling.

Since the hotness of data is defined at the application level, the FTL has no way of knowing how much of cold and hot data is contained within a single page. A way to improve performance in SSDs is to split cold and hot data as much as possible into separate pages, which will make the job of the garbage collector easier [[8]](https://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/#ref).

#### Split cold and hot data

Hot data is data that changes frequently, and cold data is data that changes infrequently. If some hot data is stored in the same page as some cold data, the cold data will be copied along every time the hot data is updated in a read-modify-write operation, and will be moved along during garbage collection for wear leveling. Splitting cold and hot data as much as possible into separate pages will make the job of the garbage collector easier.

#### Buffer hot data

Extremely hot data should be buffered as much as possible and written to the drive as infrequently as possible.

#### Invalidate obsolete data in large batches

When some data is no longer needed or need to be deleted, it is better to wait and invalidate it in a large batches in a single operation. This will allow the garbage collector process to handle larger areas at once and will help minimizing internal fragmentation.

## What’s next

Part 4 is available [here](http://codecapsule.com/2014/02/12/coding-for-ssds-part-4-advanced-functionalities-and-internal-parallelism/). You can also go to the [Table of Content](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/) for this series of articles, and if you’re in a rush, you can also directly go to [Part 6](http://codecapsule.com/2014/02/12/coding-for-ssds-part-6-a-summary-what-every-programmer-should-know-about-solid-state-drives/), which is summarizing the content from all the other parts.

## References

[1] [Understanding Intrinsic Characteristics and System Implications of Flash Memory based Solid State Drives, Chen et al., 2009](http://www.cse.ohio-state.edu/hpcs/WWW/HTML/publications/papers/TR-09-2.pdf)
[2] [Parameter-Aware I/O Management for Solid State Disks (SSDs), Kim et al., 2012](http://csl.skku.edu/papers/CS-TR-2010-329.pdf)
[3] [Essential roles of exploiting internal parallelism of flash memory based solid state drives in high-speed data processing, Chen et al, 2011](http://bit.csc.lsu.edu/~fchen/paper/papers/hpca11.pdf)
[4] [Exploring and Exploiting the Multilevel Parallelism Inside SSDs for Improved Performance and Endurance, Hu et al., 2013](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=6165265)
[5] [Design Tradeoffs for SSD Performance, Agrawal et al., 2008](http://research.microsoft.com/pubs/63596/usenix-08-ssd.pdf)
[6] [Design Patterns for Tunable and Efficient SSD-based Indexes, Anand et al., 2012](http://instartlogic.com/resources/research_papers/aanand-tunable_efficient_ssd_indexes.pdf)
[7] [BPLRU: A Buffer Management Scheme for Improving Random Writes in Flash Storage, Kim et al., 2008](https://www.usenix.org/legacy/events/fast08/tech/full_papers/kim/kim.pdf)
[8] [SFS: Random Write Considered Harmful in Solid State Drives, Min et al., 2012](https://www.usenix.org/legacy/event/fast12/tech/full_papers/Min.pdf)
[9] [A Survey of Flash Translation Layer, Chung et al., 2009](http://idke.ruc.edu.cn/people/dazhou/Papers/AsurveyFlash-JSA.pdf)
[10] [A Reconfigurable FTL (Flash Translation Layer) Architecture for NAND Flash-Based Applications, Park et al., 2008](http://idke.ruc.edu.cn/people/dazhou/Papers/a38-park.pdf)
[11] [Reliably Erasing Data From Flash-Based Solid State Drives, Wei et al., 2011](https://www.usenix.org/legacy/event/fast11/tech/full_papers/Wei.pdf)
[12] [http://en.wikipedia.org/wiki/Solid-state_drive](http://en.wikipedia.org/wiki/Solid-state_drive)
[13] [http://en.wikipedia.org/wiki/Write_amplification](http://en.wikipedia.org/wiki/Write_amplification)
[14] [http://en.wikipedia.org/wiki/Flash_memory](http://en.wikipedia.org/wiki/Flash_memory)
[15] [http://en.wikipedia.org/wiki/Serial_ATA](http://en.wikipedia.org/wiki/Serial_ATA)
[16] [http://en.wikipedia.org/wiki/Trim_(computing)](http://en.wikipedia.org/wiki/Trim_(computing))
[17] [http://en.wikipedia.org/wiki/IOPS](http://en.wikipedia.org/wiki/IOPS)
[18] [http://en.wikipedia.org/wiki/Hard_disk_drive](http://en.wikipedia.org/wiki/Hard_disk_drive)
[19] [http://en.wikipedia.org/wiki/Hard_disk_drive_performance_characteristics](http://en.wikipedia.org/wiki/Hard_disk_drive_performance_characteristics)
[20] [http://centon.com/flash-products/chiptype](http://centon.com/flash-products/chiptype)
[21] [http://www.thessdreview.com/our-reviews/samsung-64gb-mlc-ssd/](http://www.thessdreview.com/our-reviews/samsung-64gb-mlc-ssd/)
[22] [http://www.anandtech.com/show/7594/samsung-ssd-840-evo-msata-120gb-250gb-500gb-1tb-review](http://www.anandtech.com/show/7594/samsung-ssd-840-evo-msata-120gb-250gb-500gb-1tb-review)
[23] [http://www.anandtech.com/show/6337/samsung-ssd-840-250gb-review/2](http://www.anandtech.com/show/6337/samsung-ssd-840-250gb-review/2)
[24] [http://www.storagereview.com/ssd_vs_hdd](http://www.storagereview.com/ssd_vs_hdd)
[25] [http://www.storagereview.com/wd_black_4tb_desktop_hard_drive_review_wd4003fzex](http://www.storagereview.com/wd_black_4tb_desktop_hard_drive_review_wd4003fzex)
[26] [http://www.storagereview.com/samsung_ssd_840_pro_review](http://www.storagereview.com/samsung_ssd_840_pro_review)
[27] [http://www.storagereview.com/micron_p420m_enterprise_pcie_ssd_review](http://www.storagereview.com/micron_p420m_enterprise_pcie_ssd_review)
[28] [http://www.storagereview.com/intel_x25-m_ssd_review](http://www.storagereview.com/intel_x25-m_ssd_review)
[29] [http://www.storagereview.com/seagate_momentus_xt_750gb_review](http://www.storagereview.com/seagate_momentus_xt_750gb_review)
[30] [http://www.storagereview.com/corsair_vengeance_ddr3_ram_disk_review](http://www.storagereview.com/corsair_vengeance_ddr3_ram_disk_review)
[31] [http://arstechnica.com/information-technology/2012/06/inside-the-ssd-revolution-how-solid-state-disks-really-work/](http://arstechnica.com/information-technology/2012/06/inside-the-ssd-revolution-how-solid-state-disks-really-work/)
[32] [http://www.anandtech.com/show/2738](http://www.anandtech.com/show/2738)
[33] [http://www.anandtech.com/show/2829](http://www.anandtech.com/show/2829)
[34] [http://www.anandtech.com/show/6489](http://www.anandtech.com/show/6489)
[35] [http://lwn.net/Articles/353411/](http://lwn.net/Articles/353411/)
[36] [http://us.hardware.info/reviews/4178/10/hardwareinfo-tests-lifespan-of-samsung-ssd-840-250gb-tlc-ssd-updated-with-final-conclusion-final-update-20-6-2013](http://us.hardware.info/reviews/4178/10/hardwareinfo-tests-lifespan-of-samsung-ssd-840-250gb-tlc-ssd-updated-with-final-conclusion-final-update-20-6-2013)
[37] [http://www.anandtech.com/show/6489/playing-with-op](http://www.anandtech.com/show/6489/playing-with-op)
[38] [http://www.ssdperformanceblog.com/2011/06/intel-320-ssd-random-write-performance/](http://www.ssdperformanceblog.com/2011/06/intel-320-ssd-random-write-performance/)
[39] [http://en.wikipedia.org/wiki/Native_Command_Queuing](http://en.wikipedia.org/wiki/Native_Command_Queuing)
[40] [http://superuser.com/questions/228657/which-linux-filesystem-works-best-with-ssd/](http://superuser.com/questions/228657/which-linux-filesystem-works-best-with-ssd/)
[41] [http://blog.superuser.com/2011/05/10/maximizing-the-lifetime-of-your-ssd/](http://blog.superuser.com/2011/05/10/maximizing-the-lifetime-of-your-ssd/)
[42] [http://serverfault.com/questions/356534/ssd-erase-block-size-lvm-pv-on-raw-device-alignment](http://serverfault.com/questions/356534/ssd-erase-block-size-lvm-pv-on-raw-device-alignment)
[43] [http://rethinkdb.com/blog/page-alignment-on-ssds/](http://rethinkdb.com/blog/page-alignment-on-ssds/)
[44] [http://rethinkdb.com/blog/more-on-alignment-ext2-and-partitioning-on-ssds/](http://rethinkdb.com/blog/more-on-alignment-ext2-and-partitioning-on-ssds/)
[45] [http://rickardnobel.se/storage-performance-iops-latency-throughput/](http://rickardnobel.se/storage-performance-iops-latency-throughput/)
[46] [http://www.brentozar.com/archive/2013/09/iops-are-a-scam/](http://www.brentozar.com/archive/2013/09/iops-are-a-scam/)
[47] [http://www.acunu.com/2/post/2011/08/why-theory-fails-for-ssds.html](http://www.acunu.com/2/post/2011/08/why-theory-fails-for-ssds.html)
[48] [http://security.stackexchange.com/questions/12503/can-wiped-ssd-data-be-recovered](http://security.stackexchange.com/questions/12503/can-wiped-ssd-data-be-recovered)
[49] [http://security.stackexchange.com/questions/5662/is-it-enough-to-only-wipe-a-flash-drive-once](http://security.stackexchange.com/questions/5662/is-it-enough-to-only-wipe-a-flash-drive-once)
[50] [http://searchsolidstatestorage.techtarget.com/feature/The-truth-about-SSD-performance-benchmarks](http://searchsolidstatestorage.techtarget.com/feature/The-truth-about-SSD-performance-benchmarks)
[51] [http://www.theregister.co.uk/2012/12/03/macronix_thermal_annealing_extends_life_of_flash_memory/](http://www.theregister.co.uk/2012/12/03/macronix_thermal_annealing_extends_life_of_flash_memory/)
[52] [http://www.eecs.berkeley.edu/~rcs/research/interactive_latency.html](http://www.eecs.berkeley.edu/~rcs/research/interactive_latency.html)
[53] [http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/](http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/)
[54] [http://www.linux-mag.com/id/8397/](http://www.linux-mag.com/id/8397/)
[55] [http://tytso.livejournal.com/2009/02/20/](http://tytso.livejournal.com/2009/02/20/)
[56] [https://wiki.debian.org/SSDOptimization](https://wiki.debian.org/SSDOptimization)
[57] [http://wiki.gentoo.org/wiki/SSD](http://wiki.gentoo.org/wiki/SSD)
[58] [https://wiki.archlinux.org/index.php/Solid_State_Drives](https://wiki.archlinux.org/index.php/Solid_State_Drives)
[59] [https://www.kernel.org/doc/Documentation/block/cfq-iosched.txt](https://www.kernel.org/doc/Documentation/block/cfq-iosched.txt)
[60] [http://www.danielscottlawrence.com/blog/should_i_change_my_disk_scheduler_to_use_NOOP.html](http://www.danielscottlawrence.com/blog/should_i_change_my_disk_scheduler_to_use_NOOP.html)
[61] [http://www.phoronix.com/scan.php?page=article&amp;item=linux_iosched_2012](http://www.phoronix.com/scan.php?page=article&item=linux_iosched_2012)
[62] [http://www.velobit.com/storage-performance-blog/bid/126135/Effects-Of-Linux-IO-Scheduler-On-SSD-Performance](http://www.velobit.com/storage-performance-blog/bid/126135/Effects-Of-Linux-IO-Scheduler-On-SSD-Performance)
[63] [http://www.axpad.com/blog/301](http://www.axpad.com/blog/301)
[64] [http://en.wikipedia.org/wiki/List_of_solid-state_drive_manufacturers](http://en.wikipedia.org/wiki/List_of_solid-state_drive_manufacturers)
[65] [http://en.wikipedia.org/wiki/List_of_flash_memory_controller_manufacturers](http://en.wikipedia.org/wiki/List_of_flash_memory_controller_manufacturers)
[66] [http://blog.zorinaq.com/?e=29](http://blog.zorinaq.com/?e=29)
[67] [http://www.gamersnexus.net/guides/956-how-ssds-are-made](http://www.gamersnexus.net/guides/956-how-ssds-are-made)
[68] [http://www.gamersnexus.net/guides/1148-how-ram-and-ssds-are-made-smt-lines](http://www.gamersnexus.net/guides/1148-how-ram-and-ssds-are-made-smt-lines)
[69] [http://www.tweaktown.com/articles/4655/kingston_factory_tour_making_of_an_ssd_from_start_to_finish/index.html](http://www.tweaktown.com/articles/4655/kingston_factory_tour_making_of_an_ssd_from_start_to_finish/index.html)
[70] [http://www.youtube.com/watch?v=DvA9koAMXR8](http://www.youtube.com/watch?v=DvA9koAMXR8)
[71] [http://www.youtube.com/watch?v=3s7KG6QwUeQ](http://www.youtube.com/watch?v=3s7KG6QwUeQ)
[72] [Understanding the Robustness of SSDs under Power Fault, Zheng et al., 2013](https://www.usenix.org/conference/fast13/technical-sessions/presentation/zheng) — [[discussion on HN]](https://news.ycombinator.com/item?id=7047118)
[73] [http://lkcl.net/reports/ssd_analysis.html](http://lkcl.net/reports/ssd_analysis.html) — [[discussion on HN]](https://news.ycombinator.com/item?id=6973179)
