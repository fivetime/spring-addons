# Maven Central 备忘录

搭建新开发环境时的个人速查表

## GPG 签名密钥
``` bash
gpg --list-keys
# 如果密钥不存在，使用以下命令生成
gpg --gen-key
# 将公钥发布到受支持的服务器之一
export GPG_PUB_KEY=（替换为 "pub" 密钥）
gpg --keyserver keyserver.ubuntu.com --send-keys $GPG_PUB_KEY
gpg --keyserver keys.openpgp.org --send-keys $GPG_PUB_KEY
gpg --keyserver pgp.mit.edu --send-keys $GPG_PUB_KEY
```


## ~/.m2/settings.xml
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <!-- OSSRH Jira 账号 -->
      <id>ossrh</id>
      <username>替换为 https://oss.sonatype.org/#profile;User%20Token 中的值</username>
      <password>替换为 https://oss.sonatype.org/#profile;User%20Token 中的值</password>
    </server>
  </servers>

  <profiles>
    <profile>
      <id>ossrh</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <gpg.executable>gpg</gpg.executable>
        <gpg.passphrase>${env.GPG_PWD}</gpg.passphrase><!-- 密码从环境变量中读取 -->
      </properties>
    </profile>
  </profiles>
</settings>
```

使用 JDK 17 发布时需要添加的 add-opens 参数：
`export JDK_JAVA_OPTIONS='--add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED --add-opens java.base/java.text=ALL-UNNAMED --add-opens java.desktop/java.awt.font=ALL-UNNAMED'`