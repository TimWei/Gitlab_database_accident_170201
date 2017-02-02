<h1>GitLab.com 数据库事故 - 2017/01/31</h1>
>> 备注: 此事故影响了数据库 (包含 issues 与 merge requests) 但无影响到 GIT Repo's (repositories 与 wikis)。

<a href='https://www.youtube.com/c/Gitlab/live'>YouTube直播 - 瞧瞧我们如何思考与解决问题</a>

* [时间轴(格林威治标准时间)](#time_line)
* [修复过程(从约17:20 UTC左右开始记录)](#rescovery)
* [遭遇之问题](#problem)
* [外部协助](#external)

<br>

<b>影响</b>
 1. 六小时左右的数据丢失
 2. 粗估有4613个普通project丶74 forks及350 imports的数据丢失，共计影响了5037个project. 由於 Git repositorie 并<b><span style='color:red'>没有</span></b>丢失，如project的User/Group於数据丢失前已存在，project将可以被重建，但project issues之类的无法重建
 3. 接近5000条留言丢失
 4. 从Kibana的记录上，粗估有707名用户活跃度受影响
 5. 於01/31 17:20之後建立的WebHooks丢失

<br>
<br>

<a id='time_line'></a>
<h3>时间轴(格林威治标准时间)</h3>

1. <b>2017/01/31 16:00/17:00 - 21:00 </b><br>	
	*	YP正在staging调适pgpool与replication，并创建了LVM快照同步production数据至staging，希望能以此快照创建其它复本，这些操作大概在数据丢失前六小时完成了。
	*	LVM快照所创建的复本并没有照着YP预期般的有用，在恶意流量与GitLab.com的高负载下，创建replication非常耗时且容易产生问题。YP需要另外一位同事的协助，但该同事该日并无上班，所以此工作中断了。

2. <b>2017/01/31 21:00 - <a href='http://monitor.gitlab.net/dashboard/db/postgres-stats?from=1485889073596&to=1485897722398'>恶意流量造成的数据库高负载</a> - <a href='https://twitter.com/gitlabstatus/status/826544148949909506'>Twitter</a> | <a href='https://www.google.com/url?q=https://gitlab.slack.com/archives/infrastructure/p1485896980015093&sa=D&ust=1486007810473000&usg=AFQjCNF21N3lG5B7SXj4PsjZAnyuuuVDwg'>Slack </b> </a><br>	
	*	依照IP地址阻挡恶意用户流量
	*	移除了一个将repository当成某种形式CDN的用户，该用户从47000个不同的IP位址登入(造成数据库高负载)，这是与设备组与技术支持组沟通後的决定。
	* 移除了恶意流量的用户 - <a href='https://www.google.com/url?q=https://gitlab.slack.com/archives/support/p1485900555017129&sa=D&ust=1486007810476000&usg=AFQjCNF7HCaFHWRZnN1jX_czG12aZF347w'>Slack</a>
	* 数据库负载回到正常，检视了PostgreSQL的状况後，排除了许多的dead tuples

3. <b>2017/01/31 22:00 - PagerDuty告警replication延迟  - <a href='https://www.google.com/url?q=https://gitlab.slack.com/archives/infrastructure/p1485899770015360&sa=D&ust=1486010949738000&usg=AFQjCNG7EOtnCQqh0OMu4RQHCg8K9uU6CQ'>Slack</a></b><br>	
	*	开始修复db2，此时db2延迟了约4GB的数据
	*	因db2.cluster同步失败，清空了/var/opt/gitlab/postgresql/data
	* db2.cluster无法连线至db1，提示max_wal_senders配置值过低，此配置用以限制WAL连线数(通常配置值=replication数量)
	* YP调整了db1的max_wal_senders至32，重新启动PostgreSQL
	* PostgreSQL 提示连线数过多，启动失败
	* YP调整max_connections，从8000改为2000，重新启动PostgreSQL (尽管配置8000已经接近一年了)
	* db2.cluster依旧同步失败，这次并无产生任何错误提示，就只是挂在那
	* 在早些时间前(大概当地时间23:00)，<a href='https://www.google.com/url?q=https://gitlab.slack.com/archives/infrastructure/p1485900765015428&sa=D&ust=1486010949748000&usg=AFQjCNGBRN6hrrrsZlKUNt0sLTEy2aW5KA'>YP就说时间太晚了，他该离线了</a>，却因为处理这些突发的同步问题而无法离线。

4. <b>2017/01/31 23:00左右</b><br>	
	*	YP想到可能是因为data文件夹存在，使得pg_basebackup出问题，决定删除整个data文件夹。几秒後他发现自己在db1.cluster.gitlab而不是db2.cluster.gitlab.com
	*	2017/01/31 23:27 YP - 中止删除，但是太迟了。310 GB左右的数据被删到只剩下4.5 GB - <a href='https://www.google.com/url?q=https://gitlab.slack.com/archives/infrastructure/p1485905230015681&sa=D&ust=1486010949752000&usg=AFQjCNFDXSsYTR3Z2dA1WgScIPcJFg8Jiw'>Slack</a>

<br>
<br>

<a id='rescovery'></a>
<h3>修复过程 - 2017/01/31 23:00 (从约17:20 UTC左右开始记录)</h3>

1. 建议修复的方法
	* 导入db1.staging.gitlab.com的数据(约六小时前的数据)
		* CW: Staging环境并没有同步WebHook
	* 重建LVM快照(约六小时前的快照)
	* Sid: 试着回覆删除的文件如何?
		* CW: 不可能!  \`rm -Rvf\` Sid: 好吧
		* JEJ: 可能太晚了，但有没有可能将硬盘快速的切换成read-only?或process正在使用遭删除文件，有可能存在遭删除文件的file descriptor，根据http://unix.stackexchange.com/a/101247/213510
		* YP: PostgreSQL并不会随时都开启这些文件，所以这办法没效；而且Azure删除数据的速度太快了，也不会留存副本。也就是说，遭删除的数据没有办法从硬盘恢复。
		* SH: 看起来db1 staging运行了独立的PostgreSQL process在gitlab_replicator文件夹。会同步db2 production的数据。但由於之前replication延迟的关系，db2在 2016-01-31 05:53就砍了，这也使得同步中止。好消息是我们可以用在这之前的备份来还原WebHook数据。

2. 采取行动
	* 2017/02/01 23:00 - 00:00: 决定将db1.staging.gitlab.com的数据导入至db1.cluster.gitlab.com(production环境)，这将会使数据回到六小时前，并且无法还原WebHooks，这是唯一能取得的快照。YP自嘲说今天最好不要让他再执行带有sudo的指令了，将复原工作交给JN处理。
	* 2017/02/01 00:36 - JN: 备份 db1.staging.gitlab.com 数据
	* 2017/02/01 00:55 - JN: 将 db1.staging.gitlab.com同步到 db1.cluster.gitlab.com
		* 从staging /var/opt/gitlab/postgresql/data/ 复制文件至production /var/opt/gitlab/postgresql/data/
	* 2017/02/01 01:05 - JN: 使用nfs-share01 服务器做为/var/opt/gitlab/db-meltdown的暂存空间
	* 2017/02/01 01:18 - JN: 复制剩馀的production环境数据，包含压缩後的pg_xlog(压缩後为 ‘20170131-db-meltodwn-backup.tar.gz’)
	* 2017/02/01 01:58 - JN: 开始从stage同步至production
	* 2017/02/01 02:00 - CW: 更新发布页面解释今日状况。 <a href='https://www.google.com/url?q=https://drive.google.com/open?id%3D0B_4wYK1qcPT1TlJGRXR0Z0poWWc&sa=D&ust=1486014565477000&usg=AFQjCNHNxf5CTXUA3YMfsRyfSe_5dteBSw'>Link</a> 
	* 2017/02/01 03:00 - AR: 同步率大概有50%了 (依照文件量)
	* 2017/02/01 04:00 - JN: 同步率大概56.4% (依照文件量)。数据传输速度变慢了，原因有两个，us-east与us-east2的网路I/O与Staging的硬盘吞吐限制(60 Mb/s)
	* 2017/02/01 07:00 - JN: 在db1 staging /var/opt/gitlab_replicator/postgresql内发现一份未过滤(pre-sanitized)的数据副本，在 us-east启动了db-crutch VM来备份该数据。 不幸的是，这个系统最多120GM RAM，无法支持production环境负载，这份副本将用来检查数据库状态与汇出WebHook
	* 2017/02/01 08:07 - JN: 数据传输太慢了: 传输进度用数据大小来看的话才42%
	* 2017/02/02 16:28 - JN: 数据传输完成
	* 2017/02/02 16:45 - 开始进入重建阶段

3. 重建阶段
	* [x] - 建立一份DB1的服务器快照 - DB2或DB3也可以 - 於16:36 UTC完成
	* [x] - 由於production的PostgreSQL版本为9.6.0，staging是使用9.6.1，升级 db1.cluster.gitlab.com的PostgreSQL至9.6.1
		* 安装8.16.3-EE.1
		* YP将 chef-noop移至chef-client (noop禁用了手动配置)
		* 16:45，YP运行 chef-client
	* [x] - 启动数据库 - 16:53 UTC
		* 监控启动状况以及是否可用
		* 检查数据备份状况
	* [x] - 更新Sentry DSN，错误讯息才不会回传至staging
	* [x] - 为了确保创建project/note的安全，对所有table的ID增加10k，使用https://gist.github.com/anonymous/23e3c0d41e2beac018c4099d45ec88f5 完成
	* [x] - 更新 Rails/Redis 快取
	* [x] - 尽可能的重建WebHooks
		* [x] 使用尚未移除WebHook备份的快照，启动Staging
		* [x] 检查是否留有WebHook
		* [x] 汇出“web_hooks”内的资料(如果有的话)
		* [x] 复制汇出结果至production
		* [x] 汇入至production数据库
	* [x] - 测试是否workers能透过rails终端连线
	* [x] - 逐步启动workers
	* [x] - 关闭发布页面
	* [x] - 用 @gitlabstatus的推特发文
	* [x] - 创建issue说明未来的计画与行动
		* https://gitlab.com/gitlab-com/infrastructure/issues/1094 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1095 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1096 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1097 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1098 
		*	https://gitlab.com/gitlab-com/infrastructure/issues/1099 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1100 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1101 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1102 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1103 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1104 
		* https://gitlab.com/gitlab-com/infrastructure/issues/1105 
	* [ ] - 替没有Project记录的Git repositories创建Project记录，并使用现存的User/Group做为repositories的命名空间
		* <del> PC - 我想弄份这些repos.的清单，我们就可以从数据库中确认了</del>已经有了
	* [ ] - 删除未知(丢失)命名空间的repositories
		* AR - 以前次的数据为基础制作脚本
	* [x] - 再次移除恶意使用者(好让他们不能再次造成问题)
		* [x] 有着47000个IP的CDN老兄

4. 数据重建後的待办事项
	* 创一个Issue去改变终端机的格式或颜色，让你能清楚的知道自己正在身处Production环境或Staging环境(红色:Production环境 黄色:Staging环境)。并默认所有使用者在命令栏显示完整的主机名称(例如: 用“db1.staging.gitlab.com”取代原本的“db1”): https://gitlab.com/gitlab-com/infrastructure/issues/1094
	* 或许禁止rm -rf掉PostgreSQL的data文件夹? 我不确定这样是否恰当，如果我们有更完善的replication後，是否还有必要这麽做。
	* 增加备份预警: 检查S3之纇的储存空间，增加曲线图来监控备份文件大小，当跌幅超过10%时会发出警告: https://gitlab.com/gitlab-com/infrastructure/issues/1095
	* 考虑新增最近一次备份成功的时间到数据库内，管理者就能轻松的检视(此为顾客建议 https://gitlab.zendesk.com/agent/tickets/58274)
	*	试着了解为何PostgreSQL会在max_connections配置为8000时发生故障，尽管他从 2016-05-13就是这个配置。此次事件中，有很大一部份令人沮丧的原因都来自这个突然发生的故障。: https://gitlab.com/gitlab-com/infrastructure/issues/1096 
	* 看看如何建立WAL archiving或PITR(point in time recovery)，这在更新失败时也相当有帮助: https://gitlab.com/gitlab-com/infrastructure/issues/1097 
	* 建立排错指南，用户之後说不定也会碰到一样的问题
	* 实验以AzCopy来对传数据中心的资料: Microsoft説这应该比rsync快多了
		*这看起来是个Windows才有的东西，然而我们并没有任何Windows专家(或是任何够熟悉，懂得如何正确测试的人)
		
<br>
<br>

<a id='problem'></a>
<h3>遭遇之问题</h3>
1. LVM快照默认24小时一次，是YP在六小时之前手动执行了一次
2. 常态备份似乎也是24小时执行一次，虽然YP也不知道到底存在哪里；根据JN的说法，备份是无效的，备份的结果只有几Bytes。
	* SH: 看起来pg_dump是失败的，PostgreSQL 9.2二进位文件取代了 9.6的二进位文件，因为omnibus需配置data/PG_VERSION为9.6，而workers上并没有这个配置文件所以使用了默认的9.2版本，没有结果也不会产生警告，而更旧的备份则可能是被Fog gem给清除了
3. Azure硬盘快照只在NFS服务器启动，在数据服务器并无启动
4. 当同步至Staging时并不会同步WebHooks，除非我们能从常态备份中拉取，不然将会丢失
5. 产replication的程序超级脆弱丶容易出错丶需依靠一些不一定的Shell脚本丶缺少文件
	* SH: 从这次事件後我们学到了如何翻新Staging的数据库，先对gitlab_replicator的文件夹做快照，更改replication设定，然後重新启动一台新的PostgreSQL服务器。
6. 我们到S3的备份显然也没有作用，整个都是空的
7. 我们并没有一个稳固的预警，来监控备份失败的状况，将会着手研究

总的来说, 我们部属了五种备援技术，没有一个可靠或是发挥大用处 => 六小时前做的备份反而还较有用

<br>
<img src='https://lh3.googleusercontent.com/BuOlWlAZZUHE3CHJNYX8iKKaE54lvRQngKxrWKZ8AZDcMa6qmswg-PnW6FuuBya1SYotkrv-p3Z4NHDPrXMUusqcmh2ZSPK-Yc8rfsjWUkpa4EgDxxvktRcjHBVouSk0nA'>

http://monitor.gitlab.net/dashboard/db/postgres-stats?panelId=10&fullscreen&from=now-24h&to=now 

<br>
<br>

<a id='external'></a>
<h3>外部协助</h3>
<h5>HugOps</h5> (将任何热心关心此事的人加入至此，推特或是任何来源)
* 关於此事的Twitter Moment: https://twitter.com/i/moments/826818668948549632 
*	Jan Lehnardt https://twitter.com/janl/status/826717066984116229
*	Buddy CI https://twitter.com/BuddyGit/status/826704250499633152
*	Kent https://twitter.com/kentchenery/status/826594322870996992 
*	Lead off https://twitter.com/LeadOffTeam/status/826599794659450881 
*	Mozair https://news.ycombinator.com/item?id=13539779 
*	Applicant https://news.ycombinator.com/item?id=13539729 
*	Scott Hanselman https://twitter.com/shanselman/status/826753118868275200 
*	Dave Long https://twitter.com/davejlong/status/826817435470815233 
*	Simon Slater https://twitter.com/skslater/status/826812158184980482
*	Zaim M Ramlan https://twitter.com/zaimramlan/status/826803347764043777 
*	Aaron Suggs https://twitter.com/ktheory/status/826802484467396610
*	danedevalcourt https://twitter.com/danedevalcourt/status/826791663943241728
*	Karl https://twitter.com/irutsun/status/826786850186608640
*	Zac Clay https://twitter.com/mebezac/status/826781796318707712
*	Tim Roberts https://twitter.com/cirsca/status/826781142581927936
*	Frans Bouma https://twitter.com/FransBouma/status/826766417332727809
*	Roshan Chhetri https://twitter.com/sai_roshan/status/826764344637616128
*	Samuel Boswell https://twitter.com/sboswell/status/826760159758262273
*	Matt Brunt https://twitter.com/Brunty/status/826755797933756416
*	Isham Mohamed https://twitter.com/Isham_M_Iqbal/status/826755614013485056
*	Adriå Galin https://twitter.com/adriagalin/status/826754540955377665
*	Jonathan Burke https://twitter.com/imprecision/status/826749556566134784
*	Christo https://twitter.com/xho/status/826748578240544768
*	Linux Australia https://twitter.com/linuxaustralia/status/826741475731976192
*	Emma Jane https://twitter.com/emmajanehw/status/826737286725455872
*	Rafael Dohms https://twitter.com/rdohms/status/826719718539194368
*	Mike San Romån https://twitter.com/msanromanv/status/826710492169269248
*	Jono Walker https://twitter.com/WalkerJono/status/826705353265983488
*	Tom Penrose https://twitter.com/TomPenrose/status/826704616402333697
*	Jon Wincus https://twitter.com/jonwincus/status/826683164676521985
*	Bill Weiss https://twitter.com/BillWeiss/status/826673719460274176
*	Alberto Grespan https://twitter.com/albertogg/status/826662465400340481
*	Wicket https://twitter.com/matthewtrask/status/826650119042957312
*	Jesse Dearing https://twitter.com/JesseDearing/status/826647439587188736
*	Franco Gilio https://twitter.com/fgili0/status/826642668994326528
*	Adam DeConinck https://twitter.com/ajdecon/status/826633522735505408
*	Luciano Facchinelli https://twitter.com/sys0wned/status/826628970150035456
*	Miguel Di Ciurcio F. https://twitter.com/mciurcio/status/826628765820321792
*	ceej https://twitter.com/ceejbot/status/826617667884769280

<h5>Stephen Frost</h5>

https://twitter.com/net_snow/status/826622954964393984 @gitlabstatus 嘿，我是PG的提交者，也是主要的贡献者之一，非常喜欢你们所做的一切，有需要我帮忙的话随时连络我，我很乐意帮忙。

<h5>Sam McLeod</h5>

Hey Sid, 很遗憾听到你遭遇的数据库/LVM问题，偶而就是会发生这些BUG，嘿，我们也证运行了相当多的PostgreSQL群集(主从架构)，然而我从你的报告中发现些事情，1. 你使用Slony - 那货是坨燃烧的狗屎, 甚至连http://howfuckedismydatabase.com都拿Slony来嘲笑，PostgreSQL就内建的二进位串流较可靠且快多了，我建议你们改用内建的。2. 没有提到连接池，还在postgresql.conf配置了上千的连线数 - 这些听起来相当糟糕且没有效率，我建议你用pg_bouncer当做连接池 - https://pgbouncer.github.io/，并将PostgreSQL的max_connection设置512-1024内，实际上当你连接数超过256时，你该考虑横向扩展，而不是垂直扩展。3. 报告中你提到了你的失效切换与备份程序有多脆弱，我们也写了一个简易的脚本，用来做PostgreSQL的失效切换，也写了一些文件，如果你有兴趣我可以提供给你，而说到备份上，我们使用pgBaRMan来处理日益繁重的备份工作，能每日备份两次都是靠BaRMan与PostgreSQL自带的pg_dump指令，异地备援是非常重要的，能保证性能与可移值性。4. 你还在用Azure?!?! 我会建议你快点离开那个垃圾桶，这个Microsoft平台有许多内部DNS丶NTP丶路由丶储存IO的问题，太荒谬了。我也听过几个恐怖故事，关於这些问题同时发生的。如果你想听更多调校PostgreSQL的建议欢迎联络我，我有非常多的经验可以分享。

>>>
Capt. McLeod
 一并问下，你的数据库大概占用磁盘多大空间? 我们是在讨论TB等级的数据而不是GB对吧? 7h 7 hours ago

>>>
Capt. McLeod
 我开源了我的失效切换与replication同步的脚本，同时我看见你们在讨论Pgpool，我不建议使用那个，试试看Pgbouncer替代吧 7h 7 hours ago

>>>
Capt. McLeod
 Pgpool 有一堆问题，我们是经过测试才弃用的 5h 5 hours ago

>>>
Capt. McLeod
 同时我想知道，既然这件事情了，我能否透过Twitter发表言论，或是其它方式来增加这次事件的透明度，我了解发生这些事情有多鸟，我们都有资讯交换至上的精神，当我第一次遇到这些事情时，我非常的紧张甚至呕吐了。 4h 4 hours ago

>>>
Sid Sijbrandij
 Hi Sam，感谢你的协助，介意我将你贴的那些放到公开文件上分享给团队其它人吗? 3m 3 minutes ago

>>>
Capt. McLeod
 你说那个失效切换的脚本? 3m 2 minutes ago

>>>
Sid Sijbrandij
 一字一句。 2m 1 minute ago
