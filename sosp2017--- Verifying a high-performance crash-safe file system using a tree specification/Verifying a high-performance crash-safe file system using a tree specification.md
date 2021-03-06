### Verifying a high-performance crash-safe file system using a tree specification
使用树specification 验证一个高性能的crash-safe的文件系统

ABSTRACT
DFSCQ是第一个为fsync和fdatasync提供精确规范的文件系统，允许应用程序实现高性能和防止crash的安全性，并提供机器检查证明其实现符合此规范。DFSCQ的规范拥有复杂优化的行为，包括日志传递写入和DFSCQ的验证规则，解决了文件系统实现中的一些常见错误，尽管优化非常的复杂。

构建DFSCQ的关键挑战是编写文件系统及其内部实现的规范，而不会暴露内部文件系统的详细信息。DFSCQ引入了一个元数据前缀规范，它捕获fsync和fdatasync的属性，这大概遵循Linux ext4的行为。本规范使用树序列的概念 - 文件系统树状态的逻辑序列 - 简要描述崩溃后的可能状态，并描述在respect元数据更新下，数据写入如何能够被re-ordered。这有助于应用程序开发人员证明自己应用程序的crash safety，避免应用程序级错误，例如忘记在文件和包含目录上调用fsync

评估表明，DFSCQ在SSD的大文件写入时达到103 MB / s，并可持续创建每秒1,618个文件的小文件。这比Linux ext4慢（大文件写入达到295 MB / s，小文件创建时为4,977个文件），但比最近经过验证的文件系统Yggdrasil和FSCQ要快得多。来自应用级基准测试的评估结果（包括SQLite上的TPC-C）反映了这些微型基准。

INTRODUCTION
**文件系统的优化很复杂，并且可能会导致很多问题**
文件系统通过实施复杂的优化来提高磁盘吞吐量，实现了高I/O性能和crash safety。 这些优化包括将写数据到缓冲推迟到持久存储器，将许多事务分组为单个I／O操作，checksumming journal entries，并在写入文件数据块时绕过预写日志。广泛使用的Linux ext4是一个I／O高效的文件系统的例子; 上述优化允许将多个写操作处理为单个I／O操作，并减少将数据刷新到磁盘的磁盘写入障碍。不幸的是，这些优化使文件系统的实现变得复杂。例如，ext4开发人员花了6年的时间才意识到，两个优化（绕过日志和日记本校验和的数据写入）可以在崩溃之后导致先前删除的数据泄露。这个错误在2014年11月被修复，禁止用户安装同时绕过数据写入和日志校验和的ext4文件系统，对Linux中的几个文件系统的全面研究也发现了一系列其他错误。

**有一点令人惊讶的是，没有精确的规范可以证明高性能文件系统的正确性，排除如上所述的错误。**例如，POSIX标准对于文件系统操作提供的安全保障是非常模糊的。特别关注的是fsync和fdatasync提供的保证，它们给了应用程序精确的控制，文件系统能够将什么数据刷新到永久存储器。不幸的是，文件系统对准确的正在刷新的数据提供不精确的承诺，实际上，对于Linux ext4文件系统，它取决于管理员在安装文件系统时指定的选项。因为精确性的缺失，应用程序，比如数据库和邮件服务器，尝试通过fsyncs和fdatasyncs来对crash safe执行创建、写、重命名的序列，可能使系统在意外的事件crash，造成数据的丢失。

特别值得一提的具有挑战性的行为是log-bypass writes（绕过缓冲区的写）。文件系统通常使用预写日志来保证更改按照原子和顺序刷新到磁盘。一个例外是数据写入：为避免将数据块写入磁盘两次（一次到日志，一次到文件块），文件系统在写入文件数据时绕过日志。由于数据块未写入日志，因此可能会根据记录的元数据更改进行重新排序，并且必须在规范中精确地捕获此重新排序。进一步的复杂性的出现使由于重新排序可能导致块重用存在下的变体（corruption）。比如，一个对文件f的bypass的写入能够修改磁盘上的块b，但是分配b到f的元数据的更新还没有刷新到磁盘上，b或许还被其他文件或目录使用，可能会由于bypass的写造成毁坏，这个规范必须排除上述情况。
本文的主要贡献：一个方法能够详细说明文件系统的行为，单纯的通过抽象文件系统树，而不暴露文件系统优化的内部细节，比如绕过日志直接写更新磁盘块。一个纯粹的基于树的规范可以保护应用程序开发人员不用必须明报和理解底层的文件系统实现细节。









备注：fsync函数同步内存中所有已修改的文件数据到储存设备。fdatasync()，linux系统调用。用来刷新数据到磁盘。 fdatasync只刷新数据到磁盘。fsync同时刷新数据和inode信息到磁盘。

EXT4是第四代扩展文件系统（英语：Fourth extended filesystem，缩写为 ext4）是Linux系统下的日志文件系统

元数据（Metadata），又称中介数据、中继数据，为描述数据的数据（data about data），主要是描述数据属性（property）的信息，用来支持如指示存储位置、历史数据、资源查找、文件记录等功能。
任何文件系统中的数据分为数据和元数据(metadata)。数据是指普通文件中的实际数据，而元数据指用来描述一个文件的特征的系统数据，诸如访问权限、文件拥有者以及文件数据块的分布信息(inode...)等等。
文件系统元数据（metadata）的更改都被保存在一份单独的日志里，当发生系统崩溃时可以根据日志正确地恢复数据。除此之外，日志使系统重新启动时不必进行文件系统的检查，从而缩短了恢复时间。