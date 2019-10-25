---
layout: post
title:  "部署平台设计实践"
categories: Architecture
description: 部署平台设计与研发实践。
keywords: deploy, 部署, 部署平台, Jenkins, Java, pipeline
---

# Platform Of Deploy

## 一、简介

### 1.1 目标

在项目研发过程中，解决繁重的应用系统部署流程，彻底放下双手。并且能够支持版本存在多种环境情况下的应用系统部署操作。

### 1.2 介绍

部署平台提供给研发团队一个部署项目应用系统的可视化平台，支持应用系统多版本、多环境、多主机实例，方便快捷的配置管理功能也提高了研发测试团队的生产效率。部署平台适合初创、中小企业快速上手，提高新产品的产出速度，缩短产出时间。部署平台可支持容器扩展。

### 1.3 特性

|             功能点             | 部署平台 |
| :----------------------------: | :------: |
|            应用管理            |    ✔     |
|            版本管理            |    ✔     |
|            环境管理            |    ✔     |
|          主机实例管理          |    ✔     |
|            配置管理            |    ✔     |
|            应用部署            |    ✔     |
|            应用重启            |    ✔     |
|         封板解封及发布         |    ✔     |
|     支持版本多环境关联卸载     |    ✔     |
|   支持环境多主机实例关联卸载   |    ✔     |
|    支持应用多种编译方式部署    |    ✔     |
| 不存在配置文件特殊字符转义异常 |    ✔     |
|         支持配置项删除         |    ✔     |
|       强制获取SVN版本库        |    ✔     |
|    强制拉取Maven仓库依赖包     |    ✔     |
|          支持部署记录          |    ✔     |

## 二、结构

### 2.1 系统架构

![部署平台系统架构图](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191024110423.jpg?x-oss-process=style/Watermark)

<center>2-1 系统架构图</center>
### 2.2 业务构件关系

涉及应用系统、配置文件、环境、版本及主机实例。

![部署平台业务构件图](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191024110445.jpg?x-oss-process=style/Watermark)

<center>2-2 业务构件图</center>
### 2.3 部署流程

![部署平台部署流程时序图](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191024110453.jpg?x-oss-process=style/Watermark)

<center>2-3 部署流程时序图</center>
## 三、技术要点

### 3.1 SSH免密登录

系统中不需要保存主机实例密码，只需在添加主机实例时授权主机免密登录即可。从而降低了主机实例泄密的风险，提高主机安全性。

![授权SSH免密登录示意图](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191024134641.jpg?x-oss-process=style/Watermark)

<center>3-1 授权SSH免密登录示意图</center>
SSH免密登录授权操作需要人为干预，管理员前往堡垒机对Jenkins所在主机及目标客户机进行授权，授权操作如下：

```shell
#重置jenkins密码
passwd jenkins
#将jenkins用户设置为可登录
usermod -s /bin/bash jenkins
#以jenkins用户登录
su jenkins
#生成密钥，无需输入密码
ssh-keygen -t rsa
#复制公钥到目标客户机,实现免密登录
ssh-copy-id -i ~/.ssh/id_rsa.pub root@[ip]
#将jenkins用户设置为不可登录
usermod -s /bin/false jenkins
#以下为验证操作
#复制文件
scp ROOT.war root@[ip]:/temp
#远程执行命令
ssh root@[ip] "df -h"
```

### 3.2 Jenkins pipeline script

部署平台下发部署指令，部署全流程由Jenkins pipeline实现。groovy语言编写pipeline脚本，部署的应用涉及多种编译方式，例如war、jar、web、nodejs。根据不同的编译方式编写不同的pipeline脚本，以下列举几种编译方式的pipeline脚本：

- 编译方式：war

```groovy
//节点,多台实例多个node节点
node {
    //主机IP
    def host = '#{host}'
    //应用目标主机部署Web服务器路径
    def appPath = '#{appPath}'
    //应用目标主机部署包路径
    def packagePath = '#{packagePath}'
    //应用目标主机配置文件路径
    def configPath = '#{configPath}'
    //Jenkins主机本地配置项存放路径
    def localConfigPath = '#{localConfigPath}'
    //代码版本库路径
    def codePath = '#{codePath}'
    //代码编译路径
    def compilePath = '#{compilePath}'
    //Jenkins管理的版本库验证ID
    def credentialsId = '#{credentialsId}'
    //环境类型：开发、测试、预发布、生产
    def envType = '#{envType}'
    //操作类型：部署、重启
    def operateType = '#{operateType}'

    //移除目标主机配置文件，拷贝部署平台生产的配置文件至目标主机
    stage('Properties') {
        sh "ssh $host \"rm -rf $configPath/*.properties\""
        sh "scp $localConfigPath/properties/*.properties $host:$configPath/"
    }
    if (operateType == 'deploy') {
        if (envType != 'PRD') {
            //拉代码至Jenkins主机
            stage('Checkout') {
                checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: "$credentialsId", depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: "$codePath"]], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
            }
            //编译打包
            stage('Build') {
                withEnv(["JAVA_HOME=${tool 'JDK'}", "PATH+MAVEN=${tool 'M3'}/bin"]) {
                    sh 'mvn clean package -U'
                }
            }
        }
    }
    //停止进程
    stage('Stop Progress') {
        def pids = sh returnStdout: true, script: "ssh $host \"ps -ef | grep $appPath | grep -v \'grep\' | awk \'{print \\\$2}\' | xargs\""
        if ("$pids".trim()) {
            sh "ssh $host \"sh $appPath/bin/shutdown.sh\""
        }
        sleep(5)
        //如果存在无法自动shutdown的进程，强制kill
        def lastPids = sh returnStdout: true, script: "ssh $host \"ps -ef | grep $appPath | grep -v \'grep\' | awk \'{print \\\$2}\' | xargs\""
        if ("$lastPids".trim()) {
            sh "ssh $host \"kill -9 $pids\""
        }
    }
    if (operateType == 'deploy') {
        //更换应用war包
        stage('Change Package') {
            sh "ssh $host \"rm -rf $packagePath/*\""
            if (compilePath == '') {
                sh "scp target/*.war $host:$packagePath/"
            } else {
                sh "scp $compilePath/target/*.war $host:$packagePath/"
            }
        }
    }
    //启动进程
    stage('Start Progress') {
        sh "ssh $host \"sh $appPath/bin/startup.sh\""
        //拉取部署日志
        sh "ssh $host \'timeout 100 tail -f $appPath/logs/catalina.out |sed /org.apache.coyote.AbstractProtocol.start[[:space:]]Starting[[:space:]]ProtocolHandler[[:space:]][[][\\\"]ajp-nio-/q\'"
    }
}

```

- 编译方式：nodejs

```groovy
node {
    def host = '#{host}'
    def packagePath = '#{packagePath}'
    def localConfigPath = '#{localConfigPath}'
    def configPath = '#{configPath}'
    def codePath = '#{codePath}'
    def appName = '#{appName}'
    def compilePath = '#{compilePath}'
    def credentialsId = '#{credentialsId}'
    def envType = '#{envType}'
    def operateType = '#{operateType}'

    if (configPath != '') {
        stage('Properties') {
            sh "ssh $host \"rm -rf $configPath/*.json\""
            sh "scp $localConfigPath/properties/*.json $host:$configPath/"
        }
    }
    if (operateType == 'deploy') {
        if (envType != 'PRD') {
            stage('Checkout') {
                checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: "$credentialsId", depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: "$codePath"]], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
    }
    if (configPath != '') {
        stage('Stop Progress') {
            def pids = sh returnStdout: true, script: "ssh $host \"pm2 pid $appName\""
            if ("$pids".trim()) {
                sh "ssh $host \"pm2 stop $appName\""
            }
        }
    }
    if (operateType == 'deploy') {
        stage('Change Package') {
            sh "ssh $host \"rm -rf $packagePath/*\""
            if (compilePath == '') {
                sh "scp -r * $host:$packagePath/"
            } else {
                sh "scp -r $compilePath/* $host:$packagePath/"
            }
            if (configPath != '') {
                sh "ssh $host \"cd $packagePath/app;npm install --production;\""
            }
        }
    }
    if (configPath != '') {
        stage('Start Progress') {
            sh "ssh $host \"pm2 start $configPath/pm2.json\""
        }
    }
}

```

### 3.3 Jenkins API

部署平台与Jenkins的通信，用了开源工具包com.offbytwo.jenkins。上述pipeline脚本借由此工具完成替换：

```java
    /**
     * 更新文件夹中任务script
     *
     * @param systemName  系统名称
     * @param versionName 版本名称
     * @param scriptXml   更新脚本
     * @throws IOException
     * @author YazidChen
     * @date 2019/08/16 0016
     */
    public static void updateJobXmlOfFolder(String systemName, String versionName, String scriptXml) throws IOException {
        log.info("【Info】:updateJobXmlOfFolder->Method Start!->[systemName:{}, versionName:{}]", systemName, versionName);
        FolderJob folderJob = new FolderJob(systemName, HOST_JENKINS + "job/" + systemName + "/");
        jenkinsServer.updateJob(folderJob, versionName, scriptXml, true);
    }
```

替换内容为xml，只需替换**#{nodes}**节点即可(即上述的pipeline脚本)：

```xml
<?xml version='1.1' encoding='UTF-8'?>
<flow-definition plugin="workflow-job@2.33">
    <actions/>
    <description></description>
    <keepDependencies>false</keepDependencies>
    <properties/>
    <definition class="org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition" plugin="workflow-cps@2.72">
        <script>#{nodes}</script>
        <sandbox>true</sandbox>
    </definition>
    <triggers/>
    <disabled>false</disabled>
</flow-definition>

```

### 3.4 实时日志

pipeline脚本中，已将实时部署日志拉取到了jenkins日志中：

```groovy
//拉取部署日志
sh "ssh $host \'timeout 100 tail -f $appPath/logs/catalina.out |sed /org.apache.coyote.AbstractProtocol.start[[:space:]]Starting[[:space:]]ProtocolHandler[[:space:]][[][\\\"]ajp-nio-/q\'"
```

接下来需要将jenkins的日志拉取到部署平台，实时展示到部署平台上：

```java
    /**
     * 部署日志输出
     *
     * @param versionId   版本ID
     * @param envId       环境ID
     * @param systemName  系统名称
     * @param versionName 版本名称
     * @throws IOException
     * @throws InterruptedException
     * @author YazidChen
     * @date 2019/09/11 0011
     */
    public static void streamConsoleOutput(int lastBuildNum, Long versionId, Long envId, String systemName,String versionName) 
        throws IOException,InterruptedException {
        FolderJob folderJob = new FolderJob(systemName, HOST_JENKINS + "job/" + systemName + "/");
        JobWithDetails job;
        final long startTime = System.currentTimeMillis();
        while (true) {
            Thread.sleep(1000);
            long currentTime = System.currentTimeMillis();
            job = jenkinsServer.getJob(folderJob, versionName);
            if (job.getLastBuild().getNumber() != lastBuildNum) {
                break;
            }
            if (currentTime > startTime + 10000) {
                break;
            }
        }
        Build build = job.getLastBuild();
        BuildWithDetails buildWithDetails = build.details();
        //WebSocket广播订阅地址
        String subscribePath = Constant.WS_SUBSCRIPTION.get(Constant.WS_KIND.T_DEPLOY) + Constant.WS_BUSINESS_TYPE.BUILD_LOG + "/" + versionId + "/" + envId;
        Map<String, Object> map = new HashMap<>();
        //封装的WebSocket消息体
        WsMessage wsMessage = new WsMessage();
        wsMessage.setBusinessType(Constant.WS_BUSINESS_TYPE.BUILD_LOG);
        //封装的WebSocket订阅用户列表
        WsUsers wsUsers = SpringContextUtils.getBean("wsUsers", WsUsers.class);
        JenkinsUtil.streamConsoleOutput(buildWithDetails, new BuildConsoleStreamListener() {
            //每秒监听到的字符串，广播给所有订阅用户
            @Override
            public void onData(String newLogChunk) {
                map.put("logString", newLogChunk);
                wsMessage.setData(map);
                wsUsers.sendToAll(subscribePath, wsMessage);
            }

            @Override
            public void finished() {
                map.put("logString", "Jenkins build finished!");
                wsUsers.sendToAll(subscribePath, wsMessage);
            }
        }, 1, 600, true);
    }
//以下修复了com.offbytwo.jenkins包中的问题
    /**
     * Stream build console output log as text using BuildConsoleStreamListener
     * Method can be used to asynchronously obtain logs for running build.
     *
     * @param listener        interface used to asynchronously obtain logs
     * @param poolingInterval interval (seconds) used to pool jenkins for logs
     * @param poolingTimeout  pooling timeout (seconds) used to break pooling in case build stuck
     * @throws InterruptedException in case of an error.
     * @throws IOException          in case of an error.
     */
    public static void streamConsoleOutput(BuildWithDetails buildWithDetails, final BuildConsoleStreamListener listener,final int poolingInterval, final int poolingTimeout, boolean crumbFlag) throws InterruptedException, IOException {
        // Calculate start and timeout
        final long startTime = System.currentTimeMillis();
        final long timeoutTime = startTime + (poolingTimeout * 1000);
        int bufferOffset = 0;
        while (true) {
            Thread.sleep(poolingInterval * 1000);

            ConsoleLog consoleLog;
            consoleLog = getConsoleOutputText(buildWithDetails, bufferOffset, crumbFlag);
            String logString = consoleLog.getConsoleLog();
            if (logString != null && !logString.isEmpty()) {
                listener.onData(logString);
            }
            if (consoleLog.getHasMoreData()) {
                bufferOffset = consoleLog.getCurrentBufferSize();
            } else {
                listener.finished();
                break;
            }
            long currentTime = System.currentTimeMillis();

            if (currentTime > timeoutTime) {
                log.warn("Pooling for build {0} for {2} timeout! Check if job stuck in jenkins",
                        buildWithDetails.getDisplayName(), buildWithDetails.getNumber());
                break;
            }
        }
    }

    /**
     * Get build console output log as text.
     * Use this method to periodically obtain logs from jenkins and skip chunks that were already received
     *
     * @param bufferOffset offset in console lo
     * @param crumbFlag    <code>true</code> or <code>false</code>.
     * @return ConsoleLog object containing console output of the build. The line separation is done by
     * {@code CR+LF}.
     * @throws IOException in case of a failure.
     */
    private static ConsoleLog getConsoleOutputText(BuildWithDetails buildWithDetails, int bufferOffset,boolean crumbFlag) throws IOException {
        List<NameValuePair> formData = new ArrayList<>();
        formData.add(new BasicNameValuePair("start", Integer.toString(bufferOffset)));
        String path = buildWithDetails.getUrl() + "logText/progressiveText";
        HttpResponse httpResponse = client.post_form_with_result(path, formData, crumbFlag);

        Header moreDataHeader = httpResponse.getFirstHeader(BuildWithDetails.MORE_DATA_HEADER);
        Header textSizeHeader = httpResponse.getFirstHeader(BuildWithDetails.TEXT_SIZE_HEADER);
        String response = EntityUtils.toString(httpResponse.getEntity());
        boolean hasMoreData = false;
        if (moreDataHeader != null) {
            hasMoreData = Boolean.TRUE.toString().equals(moreDataHeader.getValue());
        }
        Integer currentBufferSize = bufferOffset;
        if (textSizeHeader != null) {
            try {
                currentBufferSize = Integer.parseInt(textSizeHeader.getValue());
            } catch (NumberFormatException e) {
                log.warn("Cannot parse buffer size for job {0} build {1}. Using current offset!",
                        buildWithDetails.getDisplayName(), buildWithDetails.getNumber());
            }
        }
        return new ConsoleLog(response, hasMoreData, currentBufferSize);
    }
```

## 参考

 本文参考以下文章及项目，在此对原作者表示感谢！ 

[Jenkins用户手册](https://jenkins.io/zh/doc/)

[java-client-api](https://github.com/jenkinsci/java-client-api)