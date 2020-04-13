---
layout: post
title: Building APKs from Slack using Circle CI and Firebase cloud functions
comments: true
related: false
permalink: building-apks-slack
author: Naman Dwivedi
image: /assets/slack_slash_command.png
excerpt: For small teams or small projects, it might not be easy to have full self hosted CI/CD pipelines for building apks. Sharing apks within the team for testing purposes might also be difficult. With a simple Slack app and CircleCI integration in the project, we can have automated apk building and anyone can trigger the build from Slack for a branch and build variant.  
---
For small teams or small projects, it might not be easy to have full self hosted CI/CD pipelines for building apks. Sharing apks within the team for testing purposes might also be difficult. With a simple Slack app and CircleCI integration in the project, we can have automated apk building and anyone can trigger the build from Slack for a branch and build variant.  

------

### `/build-apk [branch] or [buildVariant]|[branch]`  
  
<img src="{{ site.baseurl }}/assets/slack_build_apk.png" alt="alt text">

------

The whole process will look like this:

* Slack app and a [slash command](https://api.slack.com/interactivity/slash-commands) that triggers the build
* Receiving POST request from the Slack app with the command details. If your project uses Firebase, this can be a simple Firebase Cloud Function.
* Triggering the build on Circle CI according to branch and build variant
* Uploading the artifcat from Circle CI back to Slack

------

### `/build-apk` slash command.

Create a new Slack app and setup a slack command as below - 


<img src="{{ site.baseurl }}/assets/slack_slash_command.png" alt="alt text">

Slack will send a HTTP POST request to the Request URL mentioned when creating the Slash command. We can enter the url for the firebase cloud function here.

------

### Firebase cloud function that handles the Slack command request and triggers Circle CI job

If your project uses Firebase, you can create the cloud function in your Android project repo itself. Run `firebase login` and `firebase init functions` in your project directory.

Let's setup the function to recieve POST request and trigger build on Circle CI using Circle CI API trigger.

Note that there are some issues in firebase cloud function working with unescaped characters due to which we are avoiding using space in the slash command. We can use pipe `|` as the separator for build variant and branch name as branch names can often have `/` or `-`.

One thing to note here is that Slack expects a response within 300ms after executing slash command or else it will show `operation_tiemout` error. Initialising the function might take more than 300ms and then waiting for Circle CI api trigger response will certaintly cross 300ms time limit. To work around this, we write the response instantly using `response.write()` and then trigger the Circle CI API in response of which we finally end the response. Note that if you instantly end the response with `res.end()` the cloud function will terminate soon after so we can't do `response.send()` or `response.end()`
before the circle ci api trigger is complete.

Also see Circle CI- [Using the API to trigger jobs](https://circleci.com/docs/2.0/api-job-trigger/)

#### src/index.ts

{% highlight typescript %}
import * as functions from 'firebase-functions';
import https = require('https');

const CIRCLECI_API_TOKEN = "your_circleci_api_token"

export const buildApk = functions.https.onRequest((request, response) => {

    const user = request.body.user_name

    let variant = 'debugRelease' // default buil variant
    let branch = 'master' // default branch

    if (request.body.text.includes('|')) {
        // both variant and branch are present
        variant = request.body.text.split("|")[0]
        branch = request.body.text.split("|")[1]
    } else {
        // only branch is present and we use default variant
        branch = request.body.text
    }

    // response to be sent back to Slack after executing command
    const responseString = `Build started by *${user}*-\nBranch: ${branch}\nVariant: ${variant}`
    
    response.writeHead(200, {'Content-type': 'application/json'})
    response.write(`{"response_type": "in_channel","text": "${responseString}"}`);

    let job = 'debug-build' // default circle ci job

    // set jobs for different variants
    if (variant == 'debugRelease') job = 'debug-release-build'
    if (variant == 'release') job = 'release-build'

    const data = JSON.stringify({
        "build_parameters": {
            "CIRCLE_JOB": job
        }
    })

    const options = {
        hostname: 'circleci.com',
        path: `/api/v1.1/project/bitbucket/org/project/tree/${branch}`,
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Content-Length': data.length,
          'Authorization': 'Basic ' + new Buffer(CIRCLECI_API_TOKEN + ':').toString('base64')
        }
    }
    const req = https.request(options, (res) => {
        console.log(`statusCode: ${res.statusCode}`)
      
        res.on('data', (d) => {
            response.end()
        })
    })
      
    req.on('error', (error) => {
          response.end()
    })
      
    req.write(data)
    req.end()
});
{% endhighlight %}

------

### Circle CI config for building APKs and uploading to Slack

#### .circleci/config.yml

{% highlight yaml %}

version: 2.1

jobs:

  build: # this can be your default job for running tests
    working_directory: ~/code
    docker:
      - image: circleci/android:api-28-alpha
        
    steps:
      - checkout
      - restore_cache:
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run: # separate step to be able to cache the dependencies depending on build.gradle
          name: Downloading Dependencies
          command: ./gradlew androidDependencies
      - save_cache:
          paths:
            - ~/.gradle
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run: 
          name: Run unit tests
          command: ./gradlew :app:testDebugUnitTest :data:testDebugUnitTest :domain:testDebugUnitTest
  

  debug-build: # job to create debug build variant
    working_directory: ~/code
    docker:
      - image: circleci/android:api-28-alpha
    steps:
      - checkout
      - restore_cache:
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run: # separate step to be able to cache the dependencies depending on build.gradle
          name: Downloading Dependencies
          command: ./gradlew androidDependencies
      - save_cache:
          paths:
            - ~/.gradle
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run: 
          name: Building APK
          command: ./gradlew :app:assembleDebug
      - run:
            name: Upload to Slack
            command: curl -F file=@$(find app/build/outputs/apk/debug -name 'app-debug*') -F channels=qa-android -F token=$SLACK_TOKEN -F filename=$(find app/build/outputs/apk/debug -name 'app-debug*' -exec basename {} \;)  https://slack.com/api/files.upload

           
  debug-release-build: # job to create debugRelease build variant
    working_directory: ~/code
    docker:
      - image: circleci/android:api-28-alpha
    steps:
      - checkout
      - restore_cache:
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run: # separate step to be able to cache the dependencies depending on build.gradle
          name: Downloading Dependencies
          command: ./gradlew androidDependencies
      - save_cache:
          paths:
            - ~/.gradle
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run:
          name: Building APK
          command: ./gradlew :app:assembleDebugRelease
      - run:
            name: Upload to Slack
            command: curl -F file=@$(find app/build/outputs/apk/debugRelease -name 'app-debugRelease*') -F channels=qa-android -F token=$SLACK_TOKEN -F filename=$(find app/build/outputs/apk/debugRelease -name 'app-debugRelease*' -exec basename {} \;)  https://slack.com/api/files.upload

{% endhighlight %}

Circle CI config will be your existing config with an extra step for uploading generated artifact to Slack. We can run following CURL to upload file to Slack -

`curl -F file=@app/build/outputs/apk/debug/app-debug.apk -F channels=qa-android -F token=$SLACK_TOKEN -F filename=app-debug.apk  https://slack.com/api/files.upload`

The Slack token can be found in the OAuth & Permissions section of your app homepage. Note that you will have to give the bot the `files:write` and `commands` permissions.

If your gradle configuration updates the output apk name according to current branch and version like `app-debug-v7.8.50-master.apk`, then you will need to slightly change the final step to find the newly built apk.

filePath = `$(find app/build/outputs/apk/debug -name 'app-debug*')`

fileName = `$(find app/build/outputs/apk/debug -name 'app-debug*' -exec basename {} \;)`

------

That's it! Now you can get the APKs by just running the `/build-apk` command from Slack.


### Notes

* You should be using a debug keystore that is included in your repository so that your APK can be signed
{% highlight gradle %}

signingConfigs {
    debug {
        keyAlias 'androiddebugkey'
        keyPassword DEBUG_KEY_PASSWORD
        storeFile file(project.rootDir.path + '/debug.keystore')
        storePassword DEBUG_KEYSTORE_PASSWORD
    }
}

buildTypes {
    debug {
        signingConfig signingConfigs.debug
    }
}

{% endhighlight %}

* For building release APK, you will probably need to put the release keystore inside your repo itself and then set the store passwords in environment variables in Circle CI. Unfortunately, it looks like Circle CI doesn't have access to environment variables if job is triggered through API and this may not be the ideal way for building release APKs

* If using firebase cloud function, you will see frequent operation_timeout error on the slash command due to Slack's 300ms response time limit, but the function would still have been executed (if there are no errors in the function itself) and the build would have been triggered on Circle CI

* All the 3 steps can be replaced with some other comibination of services. Instead of Circle CI, you might use Travis CI which should also have similar API triggers. Instead of Slack to initiate build, you can have some other form of trigger that can directly call Circle CI API to start the build.

--------


**Posted by `Naman Dwivedi` on `13 Apr 2020`**

**Tags-   `Android` ,`Circle CI`, `Slack`, `APK`**
