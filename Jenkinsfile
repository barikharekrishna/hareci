//Environment Variables for Project Level
def GITPATH="https://gitlab.com/debasisroutray143/cicdpipeline.git"
def GROUP_EMAIL="debasis.routray@servion.com"

// SonarQube Address
def Sonar_Scanner = "C:\\ProgramData\\Jenkins\\.jenkins\\tools\\hudson.plugins.sonar.MsBuildSQRunnerInstallation\\sonar-ms\\sonar-scanner-4.7.0.2747"
def SONAR_ID = "sonar"

//Nexus Server Location
def NEXUS_VERSION = "nexus3"
def NEXUS_PROTOCOL = "http"
def NEXUS_URL = "192.168.99.4:8081"
def NEXUS_REPOSITORY = "maven-release"
def NEXUS_REPO_ID    = "maven-release"
def NEXUS_CREDENTIAL_ID = "nexus"
def ARTVERSION = "${env.BUILD_ID}"

// Tools Details
def GIT = "mygit"
def MAVEN = "maven3.8.6"
def JDK = "JAVA11.0.16"

pipeline {

    agent { label "master" }

tools {
  git "$GIT"
  maven "$MAVEN"
  jdk "$JDK"
}

    environment {


        PATH = "C:\\WINDOWS\\SYSTEM32"
        EMAIL_TO = 'debasis.routray@servion.com'

    }

    stages{
    
        stage('Fetch Code') {
            steps {
                git branch: 'master', url: "${GITPATH}"
            }
        }
        
        stage('BUILD'){
            steps {
                bat 'mvn clean install  '
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Code Analysis with Checkstyle'){
            steps {
                bat 'mvn checkstyle:checkstyle'
            }
    }
        
        stage('Code Analysis with SONARQUBE') {

            environment {
             
                scannerHome = tool 'sonar-ms'
                }
            tools {
                jdk 'JAVA11.0.16'
            }
            steps {
                script {
                // Import Variables from project level to stage.
                        env.SONAR_PATH="$Sonar_Scanner"
                }
                echo  "${env.scannerHome}"
                withSonarQubeEnv("$SONAR_ID") {
                    bat '''
                    %SONAR_PATH%\\bin\\sonar-scanner \
                    -Dsonar.projectKey=test \
                    -Dsonar.projectName=app \
                    -Dsonar.java.binaries=target/classes \
                    -Dsonar.exclusions=Test-Report.html \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml \
                    -Dsonar.issuesReport.html.enable=true'''
                }
                echo 'Sonarqube Executed'
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }

        post {
            always {
                httpRequest(
                    url: "http://192.168.56.3:9000/api/measures/component_tree?ps=100&s=qualifier,name&component=test&metricKeys=ncloc,bugs,vulnerabilities,code_smells,security_hotspots,coverage,duplicated_lines_density&strategy=children",
                    authentication: 'sonar-id',
                    outputFile: 'target/SonarQube-Output.json'
                )
            }
        }
    }
    
        
        stage('Performing Tests') {
                steps {
                    bat 'mvn verify'
                }
                post {
                    always {
                        echo 'Archiving Junit Report'
                        junit 'target/**/*.xml'
                }
            }
        
        }
        stage('Generate HTML-Report') {
                steps {
                    bat 'mvn site'
                }
                post {
                    always {
                        bat "copy target\\site\\surefire-report.html Test-Report.html"
                        zip zipFile: 'target/test.zip', archive: false, dir: 'target/site'
            }
        }        
    }
        


        stage("Publish to Nexus Repository Manager") {
                steps {
                    script {
                        pom = readMavenPom file: "pom.xml";
                        filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                        echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                        artifactPath = filesByGlob[0].path;
                        artifactExists = fileExists artifactPath;
                        if(artifactExists) {
                            echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION";
                            nexusArtifactUploader(
                                nexusVersion: NEXUS_VERSION,
                                protocol: "${NEXUS_PROTOCOL}",
                                nexusUrl: "${NEXUS_URL}",
                                groupId: pom.groupId,
                                version: "${ARTVERSION}",
                                repository: "${NEXUS_REPOSITORY}",
                                credentialsId: "${NEXUS_CREDENTIAL_ID}",
                                artifacts: [
                                    [artifactId: pom.artifactId,
                                    classifier: '',
                                    file: artifactPath,
                                    type: pom.packaging],
                                    [artifactId: pom.artifactId,
                                    classifier: '',
                                    file: "pom.xml",
                                    type: "pom"]
                                ]
                            );
                        } 
    		    else {
                            error "*** File: ${artifactPath}, could not be found";
                        }
                    }
                }
            }           
        }
        post {
          always {
                echo 'Executing Post Actions'
                emailext attachLog: true, attachmentsPattern: 'target/SonarQube-Output.json,Test-Report.html,target/site/checkstyle.html,target/test.zip', body: '''${SCRIPT, template="groovy-html.template"},${FILE,path="target/SonarQube-output.json"},${FILE,path="Test-Report.html"},${FILE,path="target/site/checkstyle.html"}''',
                mimeType: 'text/html',
                subject: "[Jenkins] ${currentBuild.fullDisplayName}",
                to: "debasis.routray@servion.com",
                replyTo: "debasis.routray@servion.com",
                recipientProviders: [[$class: 'CulpritsRecipientProvider']]
        }   
    }
}
