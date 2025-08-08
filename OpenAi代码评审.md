[TOC]



# 1. 流程

该项目执行流程如下：

![image-20250807142556972](https://xua-01.oss-cn-beijing.aliyuncs.com/img/image-20250807142556972.png)

**1.用户发起Push或者PR至远程仓库**

在项目文件中，需要有`./github/workflows/xxx.yml`文件,用于让Github Actions执行，当远程仓库发生属于actions监听范围内的变动时，Github检查 `.github/workflows/` 目录下所有 yml文件，解析文件中的 `on` 字段，判断**当前事件是否匹配**触发条件，若匹配，则启动 workflow 运行；否则忽略



`.github/workflows/main-remote-jar.yml` 为项目中的workflow文件,下面为触发条件

```yaml
# GitHub Actions 触发器， 当push或pull_request事件发生时
# 运行此工作，分支为‘*’，表示监测所有分支
on:
  push:
    branches:
      - '*'
  pull_request:
    branches:
      - '*
```



**2.Actions初始化**

在yml文件中，`jobs`字段表示需要执行的指令，首先将最近两次commit的代码保存下来**(保存源代码而不是差异信息)**，然后设置jdk及配置环境，再将打包好的jar包下载到actions的环境中，执行jar包中的逻辑

```yaml
jobs:
  build:
    runs-on: ubuntu-latest  # 使用ubuntu系统

    steps:  # 执行步骤
      - name: Checkout repository   # 将最近2次commit的代码保存下来(保存源代码而不是差异信息)
        uses: actions/checkout@v2
        with:
          fetch-depth: 2

      - name: Set up JDK 11   # 配置JDK版本
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt'
          java-version: '11'
```

下载打包好的jar包并执行:

```sh
# 将打包好的jar包下载至./libs/
- name: Download openai-code-review-sdk JAR
  run: wget -O ./libs/xxx.jar https://github.com/xxx.jar
  
# 执行逻辑
- name: Run Code Review
  run: java -jar ./libs/xxx.jar
        
```



**3.获取Diff Code**

首先通过`git log`命令获取最新commit记录的哈希值，然后使用`git diff hash^ hash`命令获得本次提交和上次提交的差异信息

```java
// 获得最新一次的提交的hash信息
ProcessBuilder logProcessBuilder = new ProcessBuilder("git", "log", "-1", "--pretty=format:%H");
logProcessBuilder.directory(new File("."));
Process logProcess = logProcessBuilder.start();
BufferedReader logReader = new BufferedReader(new InputStreamReader(logProcess.getInputStream()));
String latestCommitHash = logReader.readLine();
```



```java
// latestCommitHash + "^" 中的 ^ 符号是Git的特殊语法，表示「父提交」
// 获取该次提交和上一次提交之间的差异
ProcessBuilder diffProcessBuilder = new ProcessBuilder("git", "diff", latestCommitHash + "^", latestCommitHash);
diffProcessBuilder.directory(new File("."));
Process diffProcess = diffProcessBuilder.start();
StringBuilder diffCode = new StringBuilder();
BufferedReader diffReader = new BufferedReader(new InputStreamReader(diffProcess.getInputStream()));
String line;
while ((line = diffReader.readLine()) != null) {
    diffCode.append(line).append("\n");
}	
```



**4.调用AI接口进行评审**

将请求封装成DTO后发送至AI的api接口，获取返回的响应对象，从中读取出具体的评审内容

```java
// Prompt
chatCompletionRequest.setMessages(new ArrayList<ChatCompletionRequestDTO.Prompt>() {
    private static final long serialVersionUID = -7988151926241837899L;

    {
        add(new ChatCompletionRequestDTO.Prompt("user", "你是一个高级编程架构师，精通各类场景方案、架构设计和编程语言请，请您根据git diff记录，对代码做出评审。代码如下:"));
        add(new ChatCompletionRequestDTO.Prompt("user", diffCode));
    }
});
```



**5.将评审结果以日志的形式保存**

首先将日志仓库clone到**action本地环境**，在本地新增评审日志文件，将得到的评审结果写入到文件中，最后提交并推送至日志远程仓库

```java
// 将代码clone到actions的环境
Git git = Git.cloneRepository()
        .setURI(githubReviewLogUri + ".git")
        .setDirectory(new File("repo"))
        .setCredentialsProvider(new UsernamePasswordCredentialsProvider(githubToken, ""))
        .call();
```

本地新增评审日志文件

```java
// 评审日志文件名
String fileName = project+"-"+branch+"-"+author + System.currentTimeMillis() + "-" + RandomStringUtils.randomNumeric(4) + ".md";
File newFile = new File(dateFolder, fileName);
// 将ai评审信息写入文件
try (FileWriter writer = new FileWriter(newFile)) {
    writer.write(recommend);
}
```

提交并推送

```java
// 提交并推送
git.add().addFilepattern(dateFolderName + "/" + fileName).call();
git.commit().setMessage("feat: add code review new file" + fileName).call();
git.push().setCredentialsProvider(new UsernamePasswordCredentialsProvider(githubToken, "")).call();
```



**6.调用公众号接口推送评审通知**

将日志文件地址和模版数据发送给微信服务器，由微信服务器进行推送消息，管理者可以在微信收到评审通知

```java
// logUrl: 日志地址  data: 要推送的模版数据
// 将日志地址和模版数据发送给微信服务器，由微信服务器推送给用户
@Override
protected void pushMessage(String logUrl) throws Exception {
    Map<String, Map<String, String>> data = new HashMap<>();
    TemplateMessageDTO.put(data, TemplateMessageDTO.TemplateKey.REPO_NAME, gitCommand.getProject());
    TemplateMessageDTO.put(data, TemplateMessageDTO.TemplateKey.BRANCH_NAME, gitCommand.getBranch());
    TemplateMessageDTO.put(data, TemplateMessageDTO.TemplateKey.COMMIT_AUTHOR, gitCommand.getAuthor());
    TemplateMessageDTO.put(data, TemplateMessageDTO.TemplateKey.COMMIT_MESSAGE, gitCommand.getMessage());
    weiXin.sendTemplateMessage(logUrl, data);
}
```









# 2. 项目介绍

第二个项目是









# 3. 项目亮点









# 4. 面经