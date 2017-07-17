# SystemdCustomService
linux发行版使用systemd，自定义service


CentOS 7继承了RHEL 7的新的特性，例如强大的systemctl，而systemctl的使用也使得以往系统服务的/etc/init.d的启动脚本的方式就此改变，也大幅提高了系统服务的运行效率。但服务的配置和以往也发生了极大的不同，说实在的，变的简单而易用了许多。有点像windows的服务，这自然是大家喜闻乐见的。

CentOS 7的服务systemctl脚本存放在：/usr/lib/systemd/，有系统（system）和用户（user）之分，像需要开机不登陆就能运行的程序，还是存在系统服务里吧，即：/usr/lib/systemd/system目录下

机器上刚装好一个nginx和tomcat，现在将它做成服务，并且设置为开机自启动。

先做个nginx的配置文件，就像写windows的注册表一样写服务定义。

设置分为三个部份

[Unit]： unit 本身的说明，以及与其他相依 daemon 的设置，包括在什么服务之后才启动此 unit 之类的设置值；

[Service], [Socket], [Timer], [Mount], [Path]..：不同的 unit type 就得要使用相对应的设置项目。我们拿的是 sshd.service 来当范本，所以这边就使用 [Service] 来设置。 这个项目内主要在规范服务启动的脚本、环境配置文件文件名、重新启动的方式等等。

[Install]：这个项目就是将此 unit 安装到哪个 target 里面去的意思！

至于配置文件内有些设置规则还是得要说明一下：

设置项目通常是可以重复的，例如我可以重复设置两个 After 在配置文件中，不过，后面的设置会取代前面的喔！因此，如果你想要将设置值归零， 可以使用类似“ After= ”的设置，亦即该项目的等号后面什么都没有，就将该设置归零了 （reset）。

如果设置参数需要有“是/否”的项目 （布林值, boolean），你可以使用 1, yes, true, on 代表启动，用 0, no, false, off 代表关闭！随你喜好选择啰！

空白行、开头为 # 或 ; 的那一行，都代表注解！

每个部份里面还有很多的设置细项，我们使用一个简单的表格来说明每个项目好了！

[Unit] 部份	
设置参数	参数意义说明
Description	就是当我们使用 systemctl list-units 时，会输出给管理员看的简易说明！当然，使用 systemctl status 输出的此服务的说明，也是这个项目！
Documentation	这个项目在提供管理员能够进行进一步的文件查询的功能！提供的文件可以是如下的数据：Documentation=http://www.... Documentation=man:sshd（8） Documentation=file:/etc/ssh/sshd_config
After	说明此 unit 是在哪个 daemon 启动之后才启动的意思！基本上仅是说明服务启动的顺序而已，并没有强制要求里头的服务一定要启动后此 unit 才能启动。 以 sshd.service 的内容为例，该文件提到 After 后面有 network.target 以及 sshd-keygen.service，但是若这两个 unit 没有启动而强制启动 sshd.service 的话， 那么 sshd.service 应该还是能够启动的！这与 Requires 的设置是有差异的喔！
Before	与 After 的意义相反，是在什么服务启动前最好启动这个服务的意思。不过这仅是规范服务启动的顺序，并非强制要求的意思。
Requires	明确的定义此 unit 需要在哪个 daemon 启动后才能够启动！就是设置相依服务啦！如果在此项设置的前导服务没有启动，那么此 unit 就不会被启动！
Wants	与 Requires 刚好相反，规范的是这个 unit 之后最好还要启动什么服务比较好的意思！不过，并没有明确的规范就是了！主要的目的是希望创建让使用者比较好操作的环境。 因此，这个 Wants 后面接的服务如果没有启动，其实不会影响到这个 unit 本身！
Conflicts	代表冲突的服务！亦即这个项目后面接的服务如果有启动，那么我们这个 unit 本身就不能启动！我们 unit 有启动，则此项目后的服务就不能启动！ 反正就是冲突性的检查啦！
接下来了解一下在 [Service] 当中有哪些项目可以使用！

[Service] 部份	
设置参数	参数意义说明
Type	说明这个 daemon 启动的方式，会影响到 ExecStart 喔！一般来说，有下面几种类型 simple：默认值，这个 daemon 主要由 ExecStart 接的指令串来启动，启动后常驻于内存中。forking：由 ExecStart 启动的程序通过 spawns 延伸出其他子程序来作为此 daemon 的主要服务。原生的父程序在启动结束后就会终止运行。 传统的 unit 服务大多属于这种项目，例如 httpd 这个 WWW 服务，当 httpd 的程序因为运行过久因此即将终结了，则 systemd 会再重新生出另一个子程序持续运行后， 再将父程序删除。据说这样的性能比较好！！oneshot：与 simple 类似，不过这个程序在工作完毕后就结束了，不会常驻在内存中。dbus：与 simple 类似，但这个 daemon 必须要在取得一个 D-Bus 的名称后，才会继续运行！因此设置这个项目时，通常也要设置 BusName= 才行！idle：与 simple 类似，意思是，要执行这个 daemon 必须要所有的工作都顺利执行完毕后才会执行。这类的 daemon 通常是开机到最后才执行即可的服务！比较重要的项目大概是 simple, forking 与 oneshot 了！毕竟很多服务需要子程序 （forking），而有更多的动作只需要在开机的时候执行一次（oneshot），例如文件系统的检查与挂载啊等等的。
EnvironmentFile	可以指定启动脚本的环境配置文件！例如 sshd.service 的配置文件写入到 /etc/sysconfig/sshd 当中！你也可以使用 Environment= 后面接多个不同的 Shell 变量来给予设置！
ExecStart	就是实际执行此 daemon 的指令或脚本程序。你也可以使用 ExecStartPre （之前） 以及 ExecStartPost （之后） 两个设置项目来在实际启动服务前，进行额外的指令行为。 但是你得要特别注意的是，指令串仅接受“指令 参数 参数...”的格式，不能接受 <, >, >>, |, & 等特殊字符，很多的 bash 语法也不支持喔！ 所以，要使用这些特殊的字符时，最好直接写入到指令脚本里面去！不过，上述的语法也不是完全不能用，亦即，若要支持比较完整的 bash 语法，那你得要使用 Type=oneshot 才行喔！ 其他的 Type 才不能支持这些字符。
ExecStop	与 systemctl stop 的执行有关，关闭此服务时所进行的指令。
ExecReload	与 systemctl reload 有关的指令行为
Restart	当设置 Restart=1 时，则当此 daemon 服务终止后，会再次的启动此服务。举例来说，如果你在 tty2 使用文字界面登陆，操作完毕后登出，基本上，这个时候 tty2 就已经结束服务了。 但是你会看到屏幕又立刻产生一个新的 tty2 的登陆画面等待你的登陆！那就是 Restart 的功能！除非使用 systemctl 强制将此服务关闭，否则这个服务会源源不绝的一直重复产生！
RemainAfterExit	当设置为 RemainAfterExit=1 时，则当这个 daemon 所属的所有程序都终止之后，此服务会再尝试启动。这对于 Type=oneshot 的服务很有帮助！
TimeoutSec	若这个服务在启动或者是关闭时，因为某些缘故导致无法顺利“正常启动或正常结束”的情况下，则我们要等多久才进入“强制结束”的状态！
KillMode	可以是 process, control-group, none 的其中一种，如果是 process 则 daemon 终止时，只会终止主要的程序 （ExecStart 接的后面那串指令），如果是 control-group 时， 则由此 daemon 所产生的其他 control-group 的程序，也都会被关闭。如果是 none 的话，则没有程序会被关闭喔！
RestartSec	与 Restart 有点相关性，如果这个服务被关闭，然后需要重新启动时，大概要 sleep 多少时间再重新启动的意思。默认是 100ms （毫秒）。
最后，再来看看那么 Install 内还有哪些项目可用？

[Install] 部份	
设置参数	参数意义说明
WantedBy	这个设置后面接的大部分是 *.target unit ！意思是，这个 unit 本身是附挂在哪一个 target unit 下面的！一般来说，大多的服务性质的 unit 都是附挂在 multi-user.target 下面！
Also	当目前这个 unit 本身被 enable 时，Also 后面接的 unit 也请 enable 的意思！也就是具有相依性的服务可以写在这里呢！
Alias	进行一个链接的别名的意思！当 systemctl enable 相关的服务时，则此服务会进行链接文件的创建！以 multi-user.target 为例，这个家伙是用来作为默认操作环境 default.target 的规划， 因此当你设置用成 default.target 时，这个 /etc/systemd/system/default.target 就会链接到 /usr/lib/systemd/system/multi-user.target 啰！
看到这里是不是觉得和window的服务更像了，连重启都为你想到了，大致的项目就有这些，接下来让我们根据上面这些数据来进行一些简易的操作吧！

#cd /usr/lib/systemd/system

#vi nginx.service

[Unit]
Description=nginx service
After=network.target
[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true
[Install]
WantedBy=multi-user.target
保存好后在任意目录下执行，先检查一下进程nginx是否已经启动了，如果启动了先停掉。
#systemctl start nginx.service

没有什么意外发生的话说明我们的服务定义写得没问题，然后再执行

#systemctl enable nginx.service

这样我们的服务就设置为开机启动了

同上

tomcat的自启动设置已经有了眉目

#vi tomcat8080.service

[Unit]
Description=tomcat on port 8080 service
After=network.target
[Service]
Type=forking
ExecStart=/home/tomcat/apache-tomcat-8.5.9/bin/startup.sh
Restart=/home/tomcat/restart_tomcat.sh
ExecStop=/home/tomcat/apache-tomcat-8.5.9/bin/shutdown.sh
PrivateTmp=true
[Install]
WantedBy=multi-user.target
因为tomcat本身没有重启的脚本，所以没有轮子我们自己造，在任意目录新建一个restart_tomcat.sh脚本
#vi /home/tomcat/restart_tomcat.sh
#!/bin/sh
#tomcat的安装路径
p='/home/tomcat/apache-tomcat-8.5.9'
work=${p}'/work/'
`rm -rf ${work}`
tomcatpath=${p}'/bin'
echo 'operate restart tomcat: '$tomcatpath
pid=`ps aux | grep $tomcatpath | grep -v grep | grep -v retomcat | awk '{print $2}'`
echo 'exist pid:'$pid
if [ -n "$pid" ]
then
{
   echo ===========shutdown================
   $tomcatpath'/shutdown.sh'
   sleep 2
   pid=`ps aux | grep $tomcatpath | grep -v grep | grep -v retomcat | awk '{print $2}'`
   if [ -n "$pid" ]
   then
    {
      sleep 2
      echo ========kill tomcat begin==============
      echo $pid
      kill -9 $pid
      echo ========kill tomcat end==============
    }
   fi
   sleep 2
   echo ===========startup.sh==============
   $tomcatpath'/startup.sh'
 }
else
echo ===========startup.sh==============
$tomcatpath'/startup.sh'
这个脚本融合关、查、杀、重启的功能，管他什么环境情况都能重启，使用前配置好tomcat的路径就行。

最后执行命令，启动服务，设置为开机启动项。

#systemctl start tomcat8080.service

#systemctl enable tomcat8080.service

最后验收服务是否启动，浏览器访问tomcat成功。
