/* 

# 插件
用到的插件 Pipeline Utility Steps，Kubernetes Plugin, kubernetes-cli-plugin
# 共享库
sandbox-share-lib为正式共享库，Dev-sandbox-share-lib为开发共享库
# 脚本逻辑
 1. 启动一个挂载了持久存储的pod作为运行环境(目前只支持3个存储，所以最大启动任务的数量为3个)
 2. 按分支更新代码
 3. 读取代码下指定路径的配置文件，根据该配置文件生成对应的k8s部署变量(pod资源大小等)
 4. 根据获取的k8s部署变量，使用helm生成yaml文件
 5. 通过kubectl指定集群进行部署
*/


@Library('sandbox-share-lib') _
import com.foo.utils.PodTemplates

def MY_JOB_LABEL = "webservice-worker-${BUILD_ID}"

env.k8sCredentialsId = deploy.getK8sCredentialsId("${AppEnvironment}")  
env.ImageDepository = deploy.getImageDepository("${AppEnvironment}")  
env.saveBuild = 'TemporaryBuild'
env.CredentialsId = "snwdevpublic-ssh"
env.myGit = 'ssh://git@gitlab001.sandboxol.cn:31006/web/pickaxe.git'
env.javaEnterPmJks = "java-enter-pm-jks"

def appList2 = ["auth-center",
            "clan-center",
            "decoration-center",
            "friend-center",
            "game-center",
            "mailbox-center",
            "msg-center",
            "pay-center",
            "user-center",
            "activity-service",
            "charmingtown-service",
            "clan-service",
            "decoration-service",
            "friend-service",
            "gamedata-service",
            "game-service",
            "mailbox-service",
            "msg-service",
            "pay-service",
            "shop-service",
            "singleton-service",
            "user-service",
            "web-service",
            "gameaide-service",
            "admin-service",
            "datareport-service",
            "editorpay-center",
            "editorpay-service",
            "editorstatistic-service",
            "gamedata-center",
            "geoinfo-service",
            "message-process",
            "ranking-service",
            "statistic-service",
            "video-service",
            "slow-service",
            "mongo-center",
			"search-center",
			"bedwar-service",
            "gateway-service",
            "gatewaynew-service",
            "gamemongo-center",
            "notice-service",
            "temporary-service",
            "stemporary-service",
            "gtemporary-service",
            "route-service",
            "backpack-center",
            "backpack-service",
            "process-service"]

properties([
    parameters ([
        choice(
          description: '选择哪个环境进行构建',
          name: 'AppEnvironment',
          choices: ['oversea-test', 'oversea-test-restore', 'oversea-vanguard']
        ),

        string(
            description: '选择在哪个代码分支',
            name: 'MyBranch',
            defaultValue: 'test',
        ),

        choice(
            description: '选择在哪个应用上进行部署',
            name: 'app',
            choices: appList2,
        ),
        text(
            name: 'release_note',
            defaultValue: 'Release Note 信息如下所示: \n \
        Bug-Fixed: \n \
        Feature-Added: ',
            description: 'Release Note的详细信息是什么 ?'
        ),

        booleanParam(
            name: 'is_deploy_all',
            defaultValue: false,
            description: '是否部署全部应用(会覆盖选择应用选项)'
        )
    ])
])

def app = "${params.app}"
def AppEnvironment = "${params.AppEnvironment}"
def MyBranch = "${params.MyBranch}"
def is_deploy_all = "${params.is_deploy_all}"
env.nameSpace = "${params.AppEnvironment}"

def deploy_run(app) {
    if ("${app}") {
             
        def slaveTemplates = new PodTemplates()
        def image = "${ImageDepository}/${app}"
        slaveTemplates.JavaBuildTemplate {
                node(POD_LABEL) {
                    stage('git clone project') {
                            try{
                                checkout([$class: 'GitSCM', branches: [[name: "refs/heads/${MyBranch}"]], doGenerateSubmoduleConfigurations: false, submoduleCfg: [], userRemoteConfigs: [[credentialsId: "${CredentialsId}", url: "${myGit}"]]])
                                // replace imageTag tag
                                script {
                                  env.gitcommitid = sh (script: 'git rev-parse --short HEAD ${GIT_COMMIT}', returnStdout: true).trim()
                                  env.imageTag = "${env.gitcommitid}"
                                }
                            }
                            catch(all){
                                 sh "rm --force ./.git/index.lock"
                                 log.error(all)
                                 throw all
                            }
                        }
                    stage('java build'){
                            log.info("${app} build")
                            container('java8') {
                            // get AppService
                                script {
                                // game-service, get service
                                    env.AppService = utils_filter.javaFilter("${app}")
                                // read application.yml
                                    appValuesYaml = file.loadValuesYaml("server/com/sandbox/${AppService}/${app}/src/main/resources/application.yml")
                                    appConfigValues = file.readAppYaml(appValuesYaml)
                                    env.managementPort = appConfigValues[0]
                                    env.port = appConfigValues[1]
                                // read deploy yaml
                                    deployValuesYaml = file.loadValuesYaml("scripts/kubernetes/${AppEnvironment}.yaml")
                                    deployConfigValues = file.readDeployYaml(deployValuesYaml,"${app}")
                                    env.AppServiceCount = deployConfigValues.count
                                    env.AppServiceCpu = deployConfigValues.cpu
                                    env.AppServiceMemory = deployConfigValues.memory
                                    env.AppServiceLimitMemory = deployConfigValues.limitMemory
                                    env.AppServiceLimitCpu = deployConfigValues.limitCpu
                                    env.AppServiceAutoscaling = deployConfigValues.autoscaling
                                    env.AppServiceAutoscalingTargetCPU = deployConfigValues.autoscalingTargetCPU
                                    env.AppServiceAutoscalingMaxReplicas = deployConfigValues.autoscalingMaxReplicas
                                    env.AppServiceAPP_PARAMETER = deployConfigValues.APP_PARAMETER
                                    }
                                withCredentials([string(credentialsId: "${javaEnterPmJks}", variable: 'enterPmJks')]) {
                                    deploy.appBuild("${app}", "${AppService}","${saveBuild}", "${enterPmJks}")
                                    }
                                }

                     }
                    stage('image build'){
                            container('docker') {
                               def DockerfileDir = 'scripts/kubernetes/Dockerfile'
                               dir("${env.WORKSPACE}/${env.saveBuild}"){
                                     utils_sh.dockerBuild("${app}", "${AppEnvironment}","${env.WORKSPACE}","${image}","${imageTag}", "${DockerfileDir}")
                                }
                               }
                     }
                    stage('image push'){
                            container('docker') {
                             withCredentials([usernamePassword(credentialsId: 'ImageDepository-snwdevpublic', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                                  container('docker') {
                                    utils_sh.dockerPush("${ImageDepository}", "${DOCKER_HUB_USER}","${DOCKER_HUB_PASSWORD}","${image}", "${imageTag}")
                                  }
                              }
                                    }
                     }
                    stage('kubernetes prepare'){
                            def managementPort = "${env.managementPort}"
                            container('helm') {
                                dir("${env.WORKSPACE}/scripts/kubernetes"){
                                    utils_sh.k8sYamlPrepare(
                                    "${app}", "${managementPort}","${port}","${ImageDepository}", "${imageTag}", "${AppEnvironment}", "${env.AppServiceCount}",
                                    "${env.AppServiceMemory}", "${env.AppServiceLimitMemory}", "${env.AppServiceCpu}","${env.AppServiceLimitCpu}","${env.gitcommitid}",
                                    "${env.AppServiceAutoscaling}", "${env.AppServiceAutoscalingTargetCPU}", "${env.AppServiceAutoscalingMaxReplicas}", "${env.AppServiceAPP_PARAMETER}"
                                    )
                                }
                            }
                     }
                    stage('kubernetes deploy'){
                            container('kubectl') {
                                dir("${env.WORKSPACE}/scripts/kubernetes"){
                                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: "${k8sCredentialsId}", namespace: '', serverUrl: '') {
                                        utils_sh.k8sDeploy("${app}", "${nameSpace}")
                                        }
                                   }
                                }
                            }
                     }
                }
        }
     else {
        log.error('app no exist')
        error "Program failed, please read logs..."
     }
}

def deploy_all(appList){
    log.info("deploy all app")
    failed = [:]

    for (i = 0; i < appList.size(); i++){
        def _app = appList[i]
        try{
            echo "${_app}"
            deploy_run("${_app}")
        }
        catch(all){
            log.error(all)
            log.error("${_app} deploy failed")
            failed.put("${_app}", 1)
        }
        //log.info("${_app}")
    }
     if (failed){
        log.error(failed)
        error ("some app deploy failed")
     }
     else{
        log.info("deploy all app sucess")
     }
}

if ("${is_deploy_all}" == 'true')
    deploy_all(appList2)
else {
    deploy_run("${app}")
}
