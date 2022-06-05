---

title: Jenkinsfile Pipeline 案例

date: 2021-07-21 14:17:18

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Jenkins' ]

categories: [ 'CI/CD' ]

---

### JenkinsFile Pipeline案例

```groovy
//判断tag是否x.x.x
def boolean isVersionTag(String tag) {
    echo "checking version tag $tag"
    if (tag == null) {
        return false
    }
    def tagMatcher = tag =~ /\d+\.\d+\.\d+/
    return tagMatcher.matches()
}
//获取当前tag
def String readCurrentTag() {
    return sh(returnStdout: true, script: "git describe --tags").trim()
}

//开始
pipeline {
    agent any
    //工具 {jenkins_url}/configureTools
    tools {
        maven "maven3.6.1"
        jdk "jdk8"
    }
    stages {
        //检出scm
        stage("checkout") {
            steps {
                checkout scm
            }
        }
        //build
        stage("build") {
            steps {
                sh "mvn clean package -Dmaven.test.skip=true"
            }
        }
        //deploy
        //test 环境
        stage("deploy test") {
            //是否满足 如果满足则执行下一步
            when {
                expression {
                    return env.GIT_BRANCH == "origin/test"
                }
            }
            steps {
                //可能需要变量，所以使用script
                script {
                    //使用publisher over ssh
                    sshPublisher(
                            sshPublisher: false,
                            failOnError: true,
                            publishers: [
                                    sshPublisherDesc(
                                            configName: "pipeline-test",
                                            verbose: true,
                                            transfers: [
                                                    sshTransfer(
                                                            sourceFiles: "target/*.jar",
                                                            removePrefix: "target",
                                                            remoteDirectory: "jenkins-pipeline",
                                                            cleanRemote: true,
                                                            execCommand: "cd /usr/local/docker/test/jenkins-pipeline \r\nls -la"
                                                    )
                                            ]
                                    )
                            ]
                    )

                }
            }
        }
        //发布线上
        stage("deploy prod") {
            when {
                expression {
                    return env.GIT_BRANCH == "origin/master" && isVersionTag(readCurrentTag())
                }
            }
            //打包成docker镜像
            steps {
                script {
                    //使用docker插件
                    docker.withRegistry("", "a349d546-aa84-41e0-94d5-4fbfe8f41939") {
                        def image = docker.build("myImage", "./docker")
                        image.push('latest')
                        image.push(readCurrentTag())
                    }
                }
            }
        }
    }
}
```
