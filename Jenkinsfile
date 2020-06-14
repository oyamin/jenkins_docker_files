import java.time.*

enum PatchLevel {
    MAJOR, MINOR, PATCH
}

class SemVer implements Serializable {

    private int major, minor, patch

    SemVer(String version) {
        def versionParts = version.tokenize('.')
        // println versionParts
        if (versionParts.size() != 3) {
            throw new IllegalArgumentException("Wrong version format - expected MAJOR.MINOR.PATCH - got ${version}")
        }
        this.major = versionParts[0].toInteger()
        this.minor = versionParts[1].toInteger()
        this.patch = versionParts[2].toInteger()
    }

    SemVer(int major, int minor, int patch) {
        this.major = major
        this.minor = minor
        this.patch = patch
    }

    SemVer bump(PatchLevel patchLevel) {
        switch (patchLevel) {
            case PatchLevel.MAJOR:
                return new SemVer(major + 1, 0, 0)
                break
            case PatchLevel.MINOR:
                return new SemVer(major, minor + 1, 0)
                break
            case PatchLevel.PATCH:
                return new SemVer(major, minor, patch + 1)
                break
        }
        return new SemVer()
    }

    String toString() {
        return "${major}.${minor}.${patch}"
    }
}

// class gitTag implements Serializable {
//     private _tags = []
//     private String _message = ''

//     def steps
//     gitTag(steps) {this.steps = steps}

//     gitTag(String message) {
//         _message = message
//         steps.sh("git pull --tag -f")
//     }

//     void add(String tag) {
//         println "add $tag"
//         _tags.push(tag)
//         println _tags
//     }

//     void apply() {
//         def k=''
//         println "apply";
//         while( _tags ) {
//             k = _tags.pop()
//             sh("(git tag -d $k || true) 2>/dev/null")
//             sh("git tag -a $k -m \"$message\"")
//             sh("git push origin :refs/tags/$tag ||true")
//         }
//         sh("git push --tags")
//     }
// }


def updateGhTag( tag, message ) {
    // println "tag $tag"
    // println "message $message"
    sh("(git tag -d $tag || true) 2>/dev/null")
    sh("git tag -a $tag -m \"$message\"")
    sh("git push origin :refs/tags/$tag ||true")
}

//================================================
// should be moved to library
//================================================

// basic definitions
def appName = 'app_test'
def docker_registry = 'reg.eng.trueaccord.com/ap-test'
def docker_prefix = '/to_be_removed'
// docker containers will be tagged: $docker_registry+$docker_prefix+/+appName:$version-$branch
// tags and necessary parameters
def author = 'apetrenko'

def tz = TimeZone.getTimeZone('UTC')
def now = new Date()
def datetime    = now.format("yyyMMdd")   + "_" + now.format("HHmmss")         //date-time tag like 20200519_143001
def datetimeutc = now.format("yyyMMdd",tz)+ "_" + now.format("HHmmss",tz)      //date-time tag UTC timezone
def rev       = ''                                                             // short git revision tag
def message   = ''                                                             // GIT commit message
def scmVars                                                                    // SCM variables (git)
def vers      = ''                                                             // Semantic version from ssm
def version   = ''                                                             // Version variable
def branchMap = ['stable': 'stable', 'qa':'qa' ]


node {
    def WORKDIR = "${env.WORKSPACE}/${env.BUILD_NUMBER}"
    def JENKINS_USER = ""
    currentBuild.result = 'SUCCESS'

    // slackSend channel: '#devops', failOnError: false, teamDomain: 'trueaccord', tokenCredentialId: 'Slack', message: "Jenkins[Eng] - Starting build $JOB_NAME [$BUILD_NUMBER]"

    // out=sh (label: 'set', returnStdout: true, script: "set")
    // echo "$out"
    // out=sh (label: 'pwd', returnStdout: true, script: "pwd")
    // echo "$out"
    // out=sh (label: 'ls-la', returnStdout: true, script: "ls -la ")
    // echo "$out"


    //  Main procedure
    try {

        stage('Checkout source code') {
            /* Checkout source code */
            scmVars = checkout scm
            echo "${scmVars}"
        }

        env.GIT_COMMIT = scmVars.GIT_COMMIT
        env.GIT_BRANCH = scmVars.GIT_BRANCH
        env.GIT_URL    = scmVars.GIT_URL

        def repoName = scmVars.GIT_URL.tokenize('/')[-1].tokenize('.')[0]

        //get version from aws ssm 0.0.1 if new application
        versIn=sh (returnStdout: true, script: "(aws ssm get-parameter \
            --name devops.jenkins.eng.versions.$appName \
            --query 'Parameter.Value' \
            --region us-west-2 ||echo '\"0.0.0\"')| \
                jq -r '.' " )

        rev      = sh (returnStdout: true, script: "git rev-parse --short    ${GIT_COMMIT}")
        message  = sh (returnStdout: true, script: "git log --format=%B -n 1 ${GIT_COMMIT}")

        echo "version In $versIn"

        version = new SemVer(versIn)

        // updating version based on commit message
        if (message.indexOf('BUMP_MAJOR') >0 ) {
            vers = version.bump(PatchLevel.MAJOR).toString()
        } else if (message.indexOf('BUMP_MINOR') >0 ) {
            vers = version.bump(PatchLevel.MINOR).toString()
        } else {
            vers = version.bump(PatchLevel.PATCH).toString()
        }

        echo "version $vers"

        stage('docker pull') {  // Stage 1. pulling docker container or another artifact
            out=sh(returnStdout: true, script: "docker pull busybox:latest")
            echo "${out}"
        }

        if ( scmVars.GIT_BRANCH != 'master' ) {
            stage('unit testing 1'){ //testing when branch is not master
                out=sh(returnStdout: true, script: "echo 'unit testing 1'")
                echo "${out}"
            }
        }

        if ( scmVars.GIT_BRANCH == 'master' ) {
            stage('unit testing 2'){ //testing when branch is not master
                out=sh(returnStdout: true, script: "echo 'unit testing 2'")
                echo "${out}"
            }
        }
        stage('Updating version'){
            update=sh (returnStdout: true, script: "aws ssm put-parameter \
            --name devops.jenkins.eng.versions.$appName \
            --value $vers \
            --overwrite \
            --type String \
            --region us-west-2" )
            echo "update: $update"
        }
        stage('tagging and pushing docker'){
            br = branchMap[scmVars.GIT_BRANCH]? branchMap[scmVars.GIT_BRANCH]:scmVars.GIT_BRANCH
            sh("docker tag busybox:latest $docker_registry$docker_prefix/$appName:v_$vers")
            sh("docker tag busybox:latest $docker_registry$docker_prefix/$appName:$datetimeutc")
            sh("docker tag busybox:latest $docker_registry$docker_prefix/$appName:$br")
            sh("docker tag busybox:latest $docker_registry$docker_prefix/$appName:build_${BUILD_NUMBER}")
            sh("docker push $docker_registry$docker_prefix/$appName")
        }

        stage('tagging github'){
            withCredentials([string(credentialsId: 'GitHub-jenkins.eng-token', variable: 'GITHUB_TOKEN')]) {
                sh("git config credential.helper '!f() { echo username=\"sa-ta-jenkins-prod\"; echo \"password=$GITHUB_TOKEN\"; };f'")
                sh("git pull --tag -f")
                updateGhTag("$datetimeutc",          "Passed jenkins.eng job $repoName by build ${BUILD_NUMBER}")
                updateGhTag("build_${BUILD_NUMBER}", "Passed jenkins.eng job $repoName by build ${BUILD_NUMBER}")
                updateGhTag("v_$vers",               "Passed jenkins.eng job $repoName by build ${BUILD_NUMBER}")
                sh("git push --tags")
            }
        }
        // stage('tagging github 2'){
        //     tt = new gitTag("Passed jenkins.eng job $repoName by build ${BUILD_NUMBER}")
        //     tt.add("$datetimeutc")
        //     tt.add("build_${BUILD_NUMBER}")
        //     tt.add("v_$vers")
        //     tt.apply();
        // }
        // stage('tagging github 2'){
        //     withCredentials([
        //         usernamePassword(   credentialsId: 'sa-ta-jenkins-prod-token',
        //                             usernameVariable: 'username',
        //                             passwordVariable: 'password')]) {

        //             print 'username=' + username + 'password=' + password
        //             print 'username.collect { it }=' + username.collect { it }
        //             print 'password.collect { it }=' + password.collect { it }
        //     }
        // }

        // stage('tagging github 3'){
        //     withCredentials([
        //         usernamePassword(   credentialsId: 'GitHub othc ',
        //                             usernameVariable: 'username',
        //                             passwordVariable: 'password')]) {

        //             print 'username=' + username + 'password=' + password
        //             print 'username.collect { it }=' + username.collect { it }
        //             print 'password.collect { it }=' + password.collect { it }
        //     }
        // }
        // stage('tagging github 3'){
        //     withCredentials([string(credentialsId: 'GitHub othc', variable: 'GITHUB_TOKEN')]) {
        //         sh("echo $GITHUB_TOKEN >token2")
        //     }
        // }


        // stage('unit testing 2'){ //testing when branch is master
        //     when { expression { return scmVars.branch == "master"} }
        //     steps {
        //         out=sh(returnStdout: true, script: "echo two")
        //         echo "${out}"
        //     }
        // }

        // echo "-=${env.BRANCH_NAME}=-"
        // if ( "${env.BRANCH_NAME}" !='master' && "${env.BRANCH_NAME}" != 'null') {
        //     currentBuild.result = 'SUCCESS'
        //     return
        // }
        // stage('two') {
        //     out=sh(returnStdout: true, script: "echo two")
        //     echo "${out}"
        // }

        // stage('three') {
        //     out=sh(returnStdout: true, script: "echo three")
        //     echo "${out}"
        // }



        // echo "datetime:    ${datetime}"
        // echo "datetimeutc: ${datetimeutc}"
        // echo "rev:         ${rev}"
        // echo "revshort:    ${revshort}"


        // def v1=version.bump(PatchLevel.MAJOR).toString()
        // def v2=version.bump(PatchLevel.MINOR).toString()
        // def v3=version.bump(PatchLevel.PATCH).toString()

        // echo "v1 = ${v1}"
        // echo "v2 = ${v2}"
        // echo "v3 = ${v3}"




        // }


        // stage ('gh tag'){


        //     // sh("git config user.email ${repositoryCommiterEmail}")
        //     // sh("git config user.name '${repositoryCommiterUsername}'")

        //     // sh "git remote set-url origin git@github.com:..."
        //     // // deletes current snapshot tag
        //     // sh "git tag -d snapshot || true"
        //     // // tags current changeset
        //     // sh "git tag -a snapshot -m \"passed CI\""
        //     // // deletes tag on remote in order not to fail pushing the new one
        //     // sh "git push origin :refs/tags/snapshot"
        //     // // pushes the tags
        //     // sh "git push --tags"

        // }

    } catch (e) {
        currentBuild.result = 'FAILURE'
        echo "EXCEPTION: $e"
        throw e
    } finally {
        def currentResult = currentBuild.result ?: 'SUCCESS'
        def previousResult = currentBuild.previousBuild?.result

        // if (previousResult != null && previousResult != currentResult) {
        //     echo 'This will run only if the state of the Pipeline has changed'
        //     echo 'For example, if the Pipeline was previously failing but is now successful'
        // }

        echo "${previousResult}"

        if ( currentResult == "SUCCESS" ) {
            // slackSend color: "good", channel: '#jenkins-alerts-devops', failOnError: false, teamDomain: 'trueaccord', tokenCredentialId: 'Slack', message: "Jenkins[Eng] - Job: ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} was successful"
        }
        else if( currentResult == "FAILURE" ) {
            // slackSend color: "danger", channel: '#jenkins-alerts-devops', failOnError: false, teamDomain: 'trueaccord', tokenCredentialId: 'Slack', message: "Jenkins[Eng] - @${author} Job: ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} was failed. <${env.BUILD_URL}|Log>"
        }
        else if( currentResult == "UNSTABLE" ) {
            // slackSend color: "warning", channel: '#jenkins-alerts-devops', failOnError: false, teamDomain: 'trueaccord', tokenCredentialId: 'Slack', message: "Jenkins[Eng] - @${author} Job: ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} was unstable. <${env.BUILD_URL}|Log>"
        }
        else {
            // slackSend color: "danger", channel: '#jenkins-alerts-devops', failOnError: false, teamDomain: 'trueaccord', tokenCredentialId: 'Slack', message: "Jenkins[Eng] - @${author} Job: ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} its result was unclear;"
        }

        //cleaning up after build.
        //sh label: 'cleanup', returnStdout: true, script: 'sudo rm -rf pnc-webapp'
    }
}


