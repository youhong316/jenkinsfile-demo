// -*- mode: groovy -*-

// Only keep some recent builds and artefacts.
properties([[$class: 'jenkins.model.BuildDiscarderProperty',
             strategy: [$class: 'LogRotator',
                        numToKeepStr: '10',
                        artifactNumToKeepStr: '5']]])

// Build starts here. We put this in a try/except/finally block so that we can
// perform post-build actions (such as Slack notifications).
caughtError = 0
try {

    // Run tests and builds on separate nodes (potentially).
    stage "Test"
    node {

        // Checkout the source of this repository. The credentials and URL are
        // stored in the configuration of the multibranch pipeline job.
        checkout scm

        timeout(time: 5, unit: "MINUTES") {
            sh("py.test complicated_library_test.py")
        }
    }

    // Execute multiple builds in parallel. Each of these targets runs on a
    // different executor. Built objects will be archived successfully, even if
    // other parallel targets fail (because failFast is False).
    //
    // Unfortunately, these objects will not be available as "last successful"
    // artefacts. It is also not possible to build certain targets
    // individually, meaning a failed job will need to be re-run in its
    // entirety to build one artefact that previously failed.
    stage "Build and archive binaries"
    parallel makeFirstTarget: {
        node {
            checkout scm
            sh("make first")
            archive([includes: "artefacts/complicated_binary_first.deb"])
        }
    }, makeSecondTarget: {
        node {
            checkout scm
            sh("make second")
            archive([includes: "artefacts/complicated_binary_second.deb"])
        }
    }, makeThirdTarget: {
        node {
            checkout scm
            sh("make third")
            archive([includes: "artefacts/complicated_binary_third.deb"])
        }
    }, failFast: false

    // If we got here, it means the build was a success.
    currentBuild.result = "SUCCESS"
}

// Postbuild stuff
catch (caughtError) {
    currentBuild.result = "FAILURE"
    caughtError = caughtError
}

finally {

    // Run this finally block in a node to define readFile.
    node {
        // Add Slack notification. We don't run this on a separate executor
        // since it's a small job. The token is stored on the Jenkins server
        // (in private).
        String slackChannel = "#general"
        String slackDomain = "jenkinsfile-demo"
        String slackToken = readFile("/var/lib/jenkins/slack-token")

        if (currentBuild.result == "SUCCESS") {
            slackSend channel: slackChannel, teamDomain: slackDomain,
                token: slackToken, color: 'good',
                message: 'Build Successful on branch ' + env.BRANCH_NAME
        } else {
            slackSend channel: slackChannel, teamDomain: slackDomain,
                token: slackToken, color: 'bad',
                message: 'Build Failed on branch ' + env.BRANCH_NAME
        }

        // // If we're on the release branch, push binaries to Azure.
        // if (currentBuild.result == "SUCCESS") {
        //     if (env.BRANCH_NAME == "release") {
        //         String azureToken = readFile("/var/lib/jenkins/azure-token")
        //         env.AZURE_STORAGE_CONNECTION_STRING = "DefaultEndpointsProtocol=https;AccountName=jenkinsfiledemo;AccountKey=" + azureToken + ";"
        //         sh('''#!/bin/bash
        //            TAG=$(git name-rev --tags --name-only HEAD | cut -f1 -d.)
        //            if [ \"$(echo $TAG | tr -d [:space:])\" == \"undefined\" ];
        //                then git rev-parse --short HEAD > save_id
        //                else echo $TAG > save_id
        //            fi
        //            ''')

        //         // Create container for this build.
        //         sh("azure storage container create \$(cat save_id)")

        //         // Save artefact as a blob in that container.
        //         sh('''#!/bin/bash
        //            for FL in \$(ls artefacts);
        //                do azure storage blob upload artefacts/\$FL \$(cat save_id) \$(echo \$FL | cut -d. -f1)_\$(cat save_id).deb")
        //            done
        //            ''')
        //     }
        // }


        // Re-throw error if one was thrown in the try block.
        if (caughtError != 0) {
            throw caughtError
        }
    }
}
