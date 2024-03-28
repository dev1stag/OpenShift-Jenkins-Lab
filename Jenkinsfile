def appName = "birthday-paradox"
def replicas = "1"
def devProject = "mfarhaouii-dev"
def testProject = devProject // Since you have only one project
def prodProject = devProject // Since you have only one project

def skopeoToken
def imageTag

def getVersionFromPom() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    matcher ? matcher[0][1] : null
}

def skopeoCopy(def skopeoToken, def srcProject, def destProject, def appName, def imageTag) {
    sh """skopeo copy --src-tls-verify=false --src-creds=jenkins:${skopeoToken} \
    --dest-tls-verify=false --dest-creds=jenkins:${skopeoToken} \
    docker://image-registry.openshift-image-registry.svc:5000/${srcProject}/${appName}:${imageTag} \
    docker://image-registry.openshift-image-registry.svc:5000/${destProject}/${appName}:${imageTag}"""
}

def deployApplication(def appName, def imageTag, def project, def replicas) {
    openshift.withCluster() {
        openshift.withProject(project) {
            dir("openshift") {
                def result = openshift.process(readFile(file:"deploy.yaml"), "-p", "APPLICATION_NAME=${appName}", "-p", "IMAGE_TAG=${imageTag}", "-p", "APPLICATION_PROJECT=${project}")
                openshift.apply(result)
            }
            openshift.selector("deployment", appName).scale("--replicas=${replicas}")
        }
    }
}
pipeline {
    agent { label "maven" }
    stages {
        stage("Setup") {
            steps {
                script {
                    // Check out the code from your source control
                    checkout scm
        
                    // Set up environment variables or initial parameters
                    imageTag = getVersionFromPom()
        
                    // Clean up the workspace to ensure the build starts clean
                    sh "rm -rf target || true"
        
                    // Logging in to OpenShift cluster, if required
                    // sh "oc login --token=${OC_TOKEN} --server=${OC_SERVER}"
        
                    // Any other preliminary setup can be added here
                }
            }
        }
        stage("Build & Test") {
            steps {
                // Example Maven build command
                sh "mvn clean package"
            }
        }
        stage("Create Image") {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(devProject) {
                            dir("openshift") {
                                // Processing and applying the build.yaml OpenShift templat
                                def result = openshift.process(readFile(file:"build.yaml"), "-p", "APPLICATION_NAME=${appName}", "-p", "IMAGE_TAG=${imageTag}")
                                openshift.apply(result)
                            }
                            dir("target") {
                                // Make sure the path and file name are correct
                                openshift.selector("bc", appName).startBuild("--from-file=birthday-paradox-${imageTag}.jar").logs("-f")
                            }
                        }
                    }
                }
            }
        }
        stage("Deploy Application to Dev") {
            steps {
                script {
                    deployApplication(appName, imageTag, devProject, replicas)
                }
            }
        }
        stage("Copy Image to Test") {
            agent { label "jenkins-agent-skopeo" }
            steps {
                script {
                    // Since it's the same project, this might be redundant, but kept for structure
                    skopeoCopy(skopeoToken, devProject, testProject, appName, imageTag)
                }
            }
        }
        stage("Deploy Application to Test") {
            steps {
                script {
                    // Again, deploying to the same 'devProject' since there's no separate test project
                    deployApplication(appName, imageTag, testProject, replicas)
                }
            }
        }
        stage("Prompt for Prod Approval") {
            steps {
                input "Deploy to prod?"
            }
        }
        stage("Copy image to Prod") {
            agent { label "jenkins-agent-skopeo" }
            steps {
                script {
                    // Redundant in a single-project setup but included for completeness
                    skopeoCopy(skopeoToken, devProject, prodProject, appName, imageTag)
                }
            }
        }
        stage("Deploy Application to Prod") {
            steps {
                script {
                    // Deploying to the same 'devProject', effectively re-deploying in the same environment
                    deployApplication(appName, imageTag, prodProject, replicas)
                }
            }
        }
    }
}
