//CI
import java.text.SimpleDateFormat
def dateFormat = new SimpleDateFormat("yyyy-MM-dd-HH-mm")
//def date = new Date()
rel_tag = dateFormat.format(new Date())
def git_branch = ""
def TRIVY_VERSION = "0.18.3"

def dockerRegistryLogin() {
    def login_command = ""
        login_command = sh(returnStdout: true,
            script: "aws ecr get-login --no-include-email --region ap-southeast-2 | sed -e 's|-e none||g'"
        )
    sh "sudo ${login_command}"
}

node {
    try{
        stage("PreBuild"){
            try{
                sh "sudo rm -rf $WORKSPACE/*"
                sh "sudo rm -rf $WORKSPACE/.env"
                dockerRegistryLogin()
                def s = env.ref
                git_branch = s.tokenize("/")[-1]
                git branch: "${git_branch}", credentialsId: "gitaccess", url: "${ssh_url}"
            }
            catch(error){
                    echo "Unable to clone the repo"
                    throw error
                    currentBuild.result = "FAILURE"
            }
        }
        stage("Build"){
            try{
                if ( "${git_branch}" == "dev" ){
                        docker_image = "957313490272.dkr.ecr.ap-southeast-2.amazonaws.com/stuff-account-management:dev-${BUILD_NUMBER}-${rel_tag}"
                } else if ( "${git_branch}" == "stage"){
                    docker_image = "957313490272.dkr.ecr.ap-southeast-2.amazonaws.com/stuff-account-management:stage-${BUILD_NUMBER}-${rel_tag}"
                } else if ( "${git_branch}" == "uat"){
                    docker_image = "957313490272.dkr.ecr.ap-southeast-2.amazonaws.com/stuff-account-management:uat-${BUILD_NUMBER}-${rel_tag}"
                } else{
                    docker_image = "957313490272.dkr.ecr.ap-southeast-2.amazonaws.com/stuff-account-management:dev-${BUILD_NUMBER}-${rel_tag}"
                }
                if ( "${git_branch}" == "dev"  || "${git_branch}" == "stage" || "${git_branch}" == "uat"){
                    sh "sudo docker build -t ${docker_image} ."
                }
            }
            catch(error){
                echo "Unable Build Docker image"
                throw error
                currentBuild.result = "FAILURE"
            }
        }
        // CVE vulnerabilites image scan
        // https://github.com/aquasecurity/trivy
        stage("TrivyScanImage") {
            try {
                // Shows only vulnerabilities which can be fixed, break if HIGH or above detected
                sh "docker run --rm \
                                -v /var/lib/jenkins/workspace-trivy:/root/.cache/ \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy:${TRIVY_VERSION} --quiet \
                                    image \
                                    --no-progress  \
                                    --ignore-unfixed \
                                    --severity HIGH,CRITICAL \
                                    --exit-code 1 \
                                    ${docker_image}"
            } catch(error) {
                echo "TrivyScanRepo Scan needs attention"

                if ( TRIVY_ALERT_BYPASS == true ) {
                  // Force success
                  currentBuild.result = "SUCCESS"
                } else {
                  throw error
                }
            }
        }
        stage("Push"){
            try{
                if ( "${git_branch}" == "dev" || "${git_branch}" == "stage" || "${git_branch}" == "uat"){
                    sh "sudo docker push ${docker_image}"
                }
                
                currentBuild.result = "SUCCESS"
            }
            catch(error){
                echo "Unable push docker image"
                throw error
                currentBuild.result = "FAILURE"
            }

        }
    }
    catch(error){
        currentBuild.result = "FAILURE"
        throw error
    }
    finally{
        sh "sudo find $WORKSPACE -mindepth 1 -delete"
    }
}
