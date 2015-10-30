import groovy.json.JsonSlurper

def uploadAppArn
def devicePoolArn

stage 'Acquire Build Node'
node('docker') {
    sh 'echo feature branch'
    //build Android app in Docker container
    stage 'Build App'
    //checkout AWS Device Farm Sample Android App from GitHub
    git 'https://github.com/kmadel/aws-device-farm-sample-app-for-android.git'
    //tell docker to pull the android skd image, 
    //start that as a container, 
    //and then run the proceeding block of workflow steps
    docker.image('kmadel/android-sdk:24.3.3').inside {
        sh './gradlew assembleDebug assembleDebugAndroidTest'
    }
    //stash successful build
    dir('app/build/outputs/apk/') {
        stash includes: '*.apk', name: 'app-test-pkg'
    }
}
//add checkpoint so if upload fails, don't need to rebuild
checkpoint 'successfully built app and test package'
//just need a generic linux node for aws steps
node('linux'){
    stage 'Set Up Device Farm Project'
    //create a new Device Farm project if projectArn paramter is not specified
    if(!projectArn) {
      writeFile file: 'create-project.json', text: '{"name": "Jenkins Workflow AWS CLI Device Farm Demo"}' 
      wrap(
        [$class: 'AmazonAwsCliBuildWrapper', 
        credentialsId: 'AWS-DEVICE-FARM-CREDS', 
        defaultRegion: 'us-west-2']) {
            sh 'aws devicefarm create-project --cli-input-json file://create-project.json > createProjectOutput'
        }
        //get project arn from output
        def createProjectOutput = readFile('createProjectOutput')
        def jsonSlurper = new JsonSlurper()
        def projectObj = jsonSlurper.parseText(createProjectOutput)
        projectArn = projectObj.project.arn
        //null jsonSluper to avoid thread issues
        jsonSlurper = null
        projectObj = null
    }
    //create device pool with just once device for demo purposes - Samsung Galaxy S6 (Verizon)
    wrap([$class: 'AmazonAwsCliBuildWrapper',
        credentialsId: 'AWS-DEVICE-FARM-CREDS', 
        defaultRegion: 'us-west-2']) {
        sh """
            aws devicefarm create-device-pool --project-arn ${projectArn} --name android-device-pool --rules '{"attribute": "ARN", "operator": "IN", "value": "[\\"arn:aws:devicefarm:us-west-2::device:9E515A6205C14AC0B6DCDBF3FC75BC3E\\"]"}' > createDevicePoolOutput
        """
    }
    //get project arn from output to list device pools
    def createDevicePoolOutput = readFile('createDevicePoolOutput')
    jsonSlurper = new JsonSlurper()
    def devicePoolObj = jsonSlurper.parseText(createDevicePoolOutput)
    devicePoolArn = devicePoolObj.devicePool.arn
    //null jsonSluper to avoid thread issues
    jsonSlurper = null
    devicePoolObj = null
    
    stage 'Create Uploads'
    unstash 'app-test-pkg'
    parallel(
        uploadApp: {
            wrap([$class: 'AmazonAwsCliBuildWrapper',
                credentialsId: 'AWS-DEVICE-FARM-CREDS', 
                defaultRegion: 'us-west-2']) {
                sh "aws devicefarm create-upload --project-arn ${projectArn} --name app-debug.apk --type ANDROID_APP > createUploadAppOutput"
            }
            //get project arn from output to list device pools
            def createUploadAppOutput = readFile('createUploadAppOutput')
            echo createUploadAppOutput
            jsonSlurper = new JsonSlurper()
            def createUploadAppObj = jsonSlurper.parseText(createUploadAppOutput)
            //null jsonSluper to avoid thread issues
            jsonSlurper = null
            uploadAppArn = createUploadAppObj.upload.arn
            uploadAppUrl = createUploadAppObj.upload.url
            createUploadAppObj = null
            //upload app with s3 from createUpload
            sh "curl -T app-debug.apk '${uploadAppUrl}'"
            waitUntil {
                //wait until upload is complete
                wrap([$class: 'AmazonAwsCliBuildWrapper',
                    credentialsId: 'AWS-DEVICE-FARM-CREDS', 
                    defaultRegion: 'us-west-2']) {
                    sh "aws devicefarm get-upload --arn ${uploadAppArn} > getUploadAppOutput"
                }
                def getUploadAppOutput = readFile('getUploadAppOutput')
                echo getUploadAppOutput
                jsonSlurper = new JsonSlurper()
                def getUploadAppObj = jsonSlurper.parseText(getUploadAppOutput)
                def uploadStatus = getUploadAppObj.upload.status
                //null jsonSluper to avoid thread issues
                jsonSlurper = null
                getUploadAppObj = null
                if(uploadStatus == 'FAILED') {
                    error 'Upload App Failed'
                }
                uploadStatus == 'SUCCEEDED'
            }
        }, uploadTests: {
            wrap([$class: 'AmazonAwsCliBuildWrapper',
                credentialsId: 'AWS-DEVICE-FARM-CREDS', 
                defaultRegion: 'us-west-2']) {
                sh "aws devicefarm create-upload --project-arn ${projectArn} --name app-debug-androidTest-unaligned.apk --type INSTRUMENTATION_TEST_PACKAGE > createUploadTestOutput"
            }
            //get project arn from output to list device pools
            def createUploadTestOutput = readFile('createUploadTestOutput')
            echo createUploadTestOutput
            jsonSlurper = new JsonSlurper()
            def createUploadTestObj = jsonSlurper.parseText(createUploadTestOutput)
            //null jsonSluper to avoid thread issues
            jsonSlurper = null
            uploadTestArn = createUploadTestObj.upload.arn
            uploadTestUrl = createUploadTestObj.upload.url
            createUploadTestObj = null
            //upload app with s3 from createUpload
            sh "curl -T app-debug-androidTest-unaligned.apk '${uploadTestUrl}'"
            waitUntil {
                //wait until upload is complete
                wrap([$class: 'AmazonAwsCliBuildWrapper',
                    credentialsId: 'AWS-DEVICE-FARM-CREDS', 
                    defaultRegion: 'us-west-2']) {
                    sh "aws devicefarm get-upload --arn ${uploadTestArn} > getUploadTestOutput"
                }
                def getUploadTestOutput = readFile('getUploadAppOutput')
                echo getUploadTestOutput
                jsonSlurper = new JsonSlurper()
                def getUploadTestObj = jsonSlurper.parseText(getUploadTestOutput)
                uploadStatus = getUploadTestObj.upload.status
                //null jsonSluper to avoid thread issues
                jsonSlurper = null
                getUploadTestObj = null
                if(uploadStatus == 'FAILED') {
                    error 'Upload Test Failed'
                }
                uploadStatus == 'SUCCEEDED'
            }
        }, failFast: true
    )
    
    stage 'Schedule Test Run'
    wrap(
      [$class: 'AmazonAwsCliBuildWrapper', 
      credentialsId: 'AWS-DEVICE-FARM-CREDS', 
      defaultRegion: 'us-west-2']) {
          sh "aws devicefarm schedule-run --project-arn ${projectArn} --app-arn ${uploadAppArn} --device-pool-arn ${devicePoolArn} --test type=INSTRUMENTATION,testPackageArn=${uploadTestArn} > scheduleRunOutput"
    }
    stash includes: 'scheduleRunOutput', name: 'scheduleRunOutput'
}

checkpoint 'successfully uploaded app and test pkg'

stage 'Get Test Results'
//instrumentation jobs take a while, so sleep job before polling for run results
sleep time: 14, unit: 'MINUTES'
//just need a generic linux node for aws steps
node('linux'){    
    unstash 'scheduleRunOutput'
    def scheduleRunOutput = readFile('scheduleRunOutput')
    echo scheduleRunOutput
    jsonSlurper = new JsonSlurper()
    def scheduleRunObj = jsonSlurper.parseText(scheduleRunOutput)
    //null jsonSluper to avoid thread issues
    jsonSlurper = null
    runArn = scheduleRunObj.run.arn
    scheduleRunObj = null
    
    def getRunOutput
    def runResult
    waitUntil {
        //wait until upload is complete
        wrap([$class: 'AmazonAwsCliBuildWrapper',
            credentialsId: 'AWS-DEVICE-FARM-CREDS', 
            defaultRegion: 'us-west-2']) {
            sh "aws devicefarm get-run --arn ${runArn} > getRunOutput"
        }
        getRunOutput = readFile('getRunOutput')
        jsonSlurper = new JsonSlurper()
        def getRunObj = jsonSlurper.parseText(getRunOutput)
        runStatus = getRunObj.run.status
        runResult = getRunObj.run.result
        //null jsonSluper to avoid thread issues
        jsonSlurper = null
        getRunObj = null
        runStatus == 'COMPLETED'
    }
    echo getRunOutput
    if(runResult != 'PASSED') {
        error "Test Run ${runResult} - see output above for details"
    }
}
