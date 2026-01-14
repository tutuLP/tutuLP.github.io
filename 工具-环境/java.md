# JavaSpring

### windows

```shell
# 下载JDK21  21.0.6
https://www.oracle.com/java/technologies/downloads/    

# 下载maven bin.zip
https://maven.apache.org/download.cgi         
```

添加maven环境变量

```
MAVEN_HOME C:\APP\apache-maven-3.9.9-bin\apache-maven-3.9.9
M2_HOME C:\APP\apache-maven-3.9.9-bin\apache-maven-3.9.9\bin
Path %MAVEN_HOME%\bin
```

换源"C:\APP\apache-maven-3.9.9-bin\apache-maven-3.9.9\conf\settings.xml"

```
  <mirrors>
   <mirror>
         <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <name>阿里云公共仓库</name>
        <url><http://maven.aliyun.com/nexus/content/groups/public></url>
    </mirror>
    <mirror>
        <id>huaweicloud</id>
        <mirrorOf>*</mirrorOf>
        <name>华为云公共仓库</name>
        <url><https://mirrors.huaweicloud.com/repository/maven/></url>
    </mirror>
    <mirror>
      <id>maven-default-http-blocker</id>
      <mirrorOf>external:http:*</mirrorOf>
      <name>Pseudo repository to mirror external repositories initially using HTTP.</name>
      <url><http://0.0.0.0/></url>
      <blocked>true</blocked>
    </mirror>
  </mirrors>
```

IDEA新建项目，Maven java 3.4.4 Jar 21 依赖：spring web

### Linux

```shell
sudo yum remove java*
sudo yum install java-21-openjdk-devel -y

wget <https://dlcdn.apache.org/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz>
sudo tar -zxvf apache-maven-3.9.9-bin.tar.gz -C /opt
sudo vi /etc/profile
export MAVEN_HOME=/opt/apache-maven-3.9.9
export PATH=$PATH:$MAVEN_HOME/bin
source /etc/profile

java -verison
mvn -version

sudo alternatives --config java  #切换java版本
```

### mac

mac下载 java  17

```
brew install --cask temurin@17
java -version
/usr/libexec/java_home -v 17	/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home

open -e ~/.zshrc

export JAVA_HOME=$(/usr/libexec/java_home -v17)
export PATH=$JAVA_HOME/bin:$PATH

source ~/.zshrc  # or ~/.bash_profile
```



```shell
# 官网下载之后双击解压
sudo mv /Users/tutu/Downloads/apache-maven-3.9.9 /Users/tutu/

# 环境变量
sudo vim ~/.bash_profile 或 ~/.zshrc
# 添加
export MAVEN_HOME=/Users/tutu/apache-maven-3.9.9
export PATH=$PATH:$MAVEN_HOME/bin

source ~/.bash_profile
mvn -version

# 换源
sudo open -e /Users/tutu/apache-maven-3.9.9/conf/settings.xml
# 添加  
<localRepository>/Users/maven/repo</localRepository>
<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <name>aliyun</name>
  <url><https://maven.aliyun.com/repository/public></url>
</mirror>
```

运行

```
sudo mkdir -p /Users/maven/repo
sudo chmod -R 775 /Users/maven/repo
mvn clean package
java -jar target/quotation-system-0.0.1-SNAPSHOT.jar
```

