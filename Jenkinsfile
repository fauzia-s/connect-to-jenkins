/*
 * Gets all changesets since last successful build of the Jenkins job.
 *
 * @param currentJobBuild The Current Build Instance
 * @return List of ChangeSets
 */
def getAllChangeSetsSinceLastSuccessfulBuild(currentBuild) {
  def allChangeSets = []
  allChangeSets.addAll(currentBuild.changeSets)
  def build = currentBuild.previousBuild

  while (build != null) {
      if (build.result == "SUCCESS")
      {
          break
      }
      allChangeSets.addAll(build.changeSets)
      build = build.previousBuild
  }
  return allChangeSets
}

def getModifiedAqueductDSLFiles(jobChangeLogSets, prefix="") {
  // Seperating DSL & Non-DSL files
  def dslFilesList = []
  def otherFilesList = []

  for (int i = 0; i < jobChangeLogSets.size(); i++) {
    def entries = jobChangeLogSets[i].items
    for (int j = 0; j < entries.length; j++) {
      def entry = entries[j]
      def files = new ArrayList(entry.affectedFiles)
      for (int k = 0; k < files.size(); k++) {
        def file = files[k]
        if (file.path.startsWith(prefix)) {
          if(file.path.endsWith(".json")) {
            dslFilesList.add(file.path)
          } else {
            otherFilesList.add(file.path)
          }
        }
      }
    }
  }
}

 /*
 * Gets modified files from a list of changesets.
 *
 * @param currentJobBuild:  current job build
 * @param prefix: Filter the list of files by prefix
 * @param sinceLastSuccessFulBuild: if history needs to be checked
 * @return List of DSL & non DSL files that were modified and contained the prefix
 */
def getModifiedFiles(currentJobBuild, prefix, sinceLastSuccessFulBuild = false) {
  def jobChangeLogSets = []
  if (sinceLastSuccessFulBuild) {
    jobChangeLogSets = getAllChangeSetsSinceLastSuccessfulBuild(currentJobBuild)
  } else {
    jobChangeLogSets = currentJobBuild.changeSets
  }
  return getModifiedAqueductDSLFiles(jobChangeLogSets, prefix)
}

/*
 * Get changes compared to master
 */

def getDiffMain(prefix) {
  def changedFiles = sh(
          script: "git diff --name-only origin/main...HEAD --diff-filter=d -- ${prefix}",
          returnStdout: true
  ).split("\n")

  def dslFilesList = []
  def otherFilesList = []

  for (int k = 0; k < changedFiles.size(); k++){
   def file = changedFiles[k]
    if (file.startsWith(prefix))
    {
      if(file.endsWith(".json"))
         {
               dslFilesList.add(file)
          }
      else {
              otherFilesList.add(file)
            }
    }
  }
  return [dslFilesList.toSet(),otherFilesList.toSet()]
}

pipeline{
    agent any

    // this section configures Jenkins options
    options {

        // only keep 10 logs for no more than 10 days
        buildDiscarder(logRotator(daysToKeepStr: '10', numToKeepStr: '10'))

        // cause the build to time out if it runs for more than 12 hours
        timeout(time: 12, unit: 'HOURS')

        // add timestamps to the log
        timestamps()
    }

    // this section configures triggers
    triggers {
          // create a cron trigger that will run the job every day at midnight
          // note that the time is based on the time zone used by the server
          // where Jenkins is running, not the user's time zone
          cron '@midnight'
    }

    // the pipeline section we all know and love: stages! :D
    stages {
        stage('Requirements') {
            steps {
                echo 'Installing requirements...'
            }
        }
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Report') {
            steps {
                echo 'Reporting....'
            }
        }
        stage('Check Build Status') {
                    steps {
                        script {

                            def branchName = env.BRANCH_NAME //Branch name: multibranch-webhook
                            def buildNumber = currentBuild.number
                            boolean firstBuild = true

                            // Get the previous build status
                            def previousBuild = build job: "${JOB_NAME}", propagate: false, wait: false  //Job name: connect-to-jenkins repo/multibranch-webhook


                            // Check if the current build is the first build for the branch
                            if (previousBuild == null) {
                                echo "This is the first build for the branch: ${branchName}"
                            } else {
                                // Check if all previous builds have failed
                                boolean allPreviousBuildsFailed = true
                                def buildIterator = previousBuild
                                while (buildIterator != null) {
                                    if (buildIterator.result == 'SUCCESS') {
                                        // allPreviousBuildsFailed = false
                                        firstBuild = false
                                        break
                                    }
                                    buildIterator = buildIterator.previousBuild
                                }

                                // if (allPreviousBuildsFailed) {
                                //     echo "All previous builds for the branch ${branchName} have failed."
                                // } else {
                                //     echo "Previous builds for the branch ${branchName} have succeeded or are still running."
                                // }
                            }
                        }
                    }
                }
        stage('Deploy to Staging'){
          when {
              allOf {
                     not { branch 'main' }
                     not { changeRequest() }
                    }
               }
            steps {
                script
                {
                echo 'Deploying to staging'
                if (firstBuild)
                   {
                     def (dslFilesList,otherFilesList) = getDiffMaster(".")

                     echo "Modified files compared to master: ${fileList}"
                     echo "Jsons: ${dslFilesList}"
                     echo "Other files: ${otherFilesList}"
                    }
                else
                {
                //   def allChangeSets = getAllChangeSetsSinceLastSuccessfulBuild(currentBuild)
                     def fileList = getModifiedFiles(currentBuild,'')

                //   echo "${currentBuild}"
                //   echo "Changesets: ${allChangeSets}"
                     echo "Modified files ${fileList}"
//                      echo "SCM: ${scm}"

                }
              }
           }
        }
        stage('Deploy to Production'){
                  when {
                         branch 'main'
                   }
            steps{
                echo 'Deploying to production.'
           }
        }
    }

    // the post section is a special collection of stages
    // that are run after all other stages have completed
    post {

        // the always stage will always be run
        always {

            // the always stage can contain build steps like other stages
            // a "steps{...}" section is not needed.
            echo "This step will run after all other steps have finished.  It will always run, even in the status of the build is not SUCCESS"
        }
    }
}
