

pipeline {
    agent  { label 'master' }
    tools {
        go 'go'
        jdk 'jdk8'
    }
  environment {
    VERSION="0.1"
    APP_NAME = "payment"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    NL_DT_TAG="app:${env.APP_NAME},environment:dev"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    NEOLOAD_ASCODEFILE="$WORKSPACE/test/neoload/payment_neoload.yaml"
    NEOLOAD_ANOMALIEDETECTIONFILE="$WORKSPACE/monspec/payment_anomalieDection.json"
    BASICCHECKURI="health"
    PAYMENTURI="paymentAuth"
    GROUP = "neotysdevopsdemo"
    COMMIT = "DEV-${VERSION}"
  }
  stages {
      stage('Checkout') {
          agent { label 'master' }
          steps {
              git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                      branch :'master'
          }
      }
    stage('Go build') {
      steps {


          sh '''
            export GOPATH=$PWD

     
           mkdir -p src/github.com/neotysdevopsdemo/payment/
           cp -R ./api src/github.com/neotysdevopsdemo/payment/
           cp -R ./main.go src/github.com/neotysdevopsdemo/payment/
           cp -R ./glide.* src/github.com/neotysdevopsdemo/payment/
           cd src/github.com/neotysdevopsdemo/payment
           go get -v github.com/Masterminds/glide
          
           
         
          '''
        }

    }
    stage('Docker build') {
        steps {
            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker pull ${GROUP}/${APP_NAME}:DEV-0.1"
                sh "docker tag ${GROUP}/${APP_NAME}:DEV-0.1 ${TAG_DEV}"
                sh "docker login --username=${USER} --password=${TOKEN}"
                sh "docker push ${TAG_DEV}"


            }

        }

    }

    stage('Deploy to dev namespace') {
        steps {
            sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},'  $WORKSPACE/docker-compose.yml"
            sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

        }

    }
   /* stage('DT Deploy Event') {
        when {
            expression {
            return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
            }
        }
        steps {
          container("curl") {
            // send custom deployment event to Dynatrace
            sh "curl -X POST \"$DT_TENANT_URL/api/v1/events?Api-Token=$DT_API_TOKEN\" -H \"accept: application/json\" -H \"Content-Type: application/json\" -d \"{ \\\"eventType\\\": \\\"CUSTOM_DEPLOYMENT\\\", \\\"attachRules\\\": { \\\"tagRule\\\" : [{ \\\"meTypes\\\" : [\\\"SERVICE\\\"], \\\"tags\\\" : [ { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"app\\\", \\\"value\\\" : \\\"${env.APP_NAME}\\\" }, { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"environment\\\", \\\"value\\\" : \\\"dev\\\" } ] }] }, \\\"deploymentName\\\":\\\"${env.JOB_NAME}\\\", \\\"deploymentVersion\\\":\\\"${_VERSION}\\\", \\\"deploymentProject\\\":\\\"\\\", \\\"ciBackLink\\\":\\\"${env.BUILD_URL}\\\", \\\"source\\\":\\\"Jenkins\\\", \\\"customProperties\\\": { \\\"Jenkins Build Number\\\": \\\"${env.BUILD_ID}\\\",  \\\"Git commit\\\": \\\"${env.GIT_COMMIT}\\\" } }\" "
          }
        }
    }*/
      stage('Start NeoLoad infrastructure') {

          steps {
              sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml up -d'

          }

      }
      stage('Join Load Generators to Application') {

          steps {
              sh 'docker network connect payment_master_default docker-lg1'
          }
      }
    stage('Run health check in dev') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=payment_master_default'
                dir 'infrastructure/infrastructure/neoload/controller/'
            }
        }
      steps {
        echo "Waiting for the service to start..."
        sleep 300
        sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's/PAYMENT_TO_REPLACE/${PAYMENTURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's,JSONFILE_TO_REPLACE,$WORKSPACE/monspec/payment_anomalieDection.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
        sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"

          script {

              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                      testName: 'HealthCheck_payment_${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'HealthCheck_payment_${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-project $WORKSPACE/test/neoload/payment_neoload.yaml -nlweb -L BasicCheck=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME},port=80",
                      scenario: 'BasicCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                              'ErrorRate'
                      ]

          }

      }
    }
    stage('Sanity Check') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=payment_master_default'
                dir 'infrastructure/infrastructure/neoload/controller/'
            }
        }
          steps {
              sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's/PAYMENT_TO_REPLACE/${PAYMENTURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's,JSONFILE_TO_REPLACE,$WORKSPACE/monspec/payment_anomalieDection.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
              sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"

              script {
                  neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                          project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                          testName: 'DynatraceSanityCheck_payment_${VERSION}_${BUILD_NUMBER}',
                          testDescription: 'DynatraceSanityCheck_Payment_${VERSION}_${BUILD_NUMBER}',
                          commandLineOption: "-project $WORKSPACE/test/neoload/payment_neoload.yaml -nlweb -L  Population_Dynatrace_SanityCheck=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -variables host=${env.APP_NAME},port=80 -nlwebToken $NLAPIKEY ",
                          scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                          trendGraphs: [
                                  [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                                  'ErrorRate'
                          ]
              }

              echo "push $WORKSPACE/infrastructure/sanitycheck.json"
              //---add the push of the sanity check---
              withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                  sh "git config --global user.email ${GIT_USERNAME}"
                  sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/payment"
                  sh "git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/*"
                  sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/payment"
              //    sh "git add $WORKSPACE/infrastructure/sanitycheck.json"
                //  sh "git commit -m 'Update Sanity_Check_${BUILD_NUMBER} ${env.APP_NAME} '"
                  //  sh "git pull -r origin master"
                  //#TODO handle this exeption
                  // sh "git push origin HEAD:master"

              }
          }
    }
    stage('Run functional check in dev') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=payment_master_default'
                dir 'infrastructure/infrastructure/neoload/controller/'
            }
        }

      steps {
         sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/PAYMENT_TO_REPLACE/${PAYMENTURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's,JSONFILE_TO_REPLACE,$WORKSPACE/monspec/payment_anomalieDection.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"

          script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                      testName: 'FuncCheck_payment_${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'FuncCheck_payment_${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-project  $WORKSPACE/test/neoload/payment_neoload.yaml -nlweb -L paymentLoad=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=payment,port=80",
                      scenario: 'paymentLoad', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                              'ErrorRate'
                      ]
          }


      }
    }
    stage('Mark artifact for staging namespace') {

      steps {

              withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                  sh "docker login --username=${USER} --password=${TOKEN}"

                  sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
                  sh "docker push ${TAG_STAGING}"
              }


      }


    }

  }
   post {
       always {
           sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
           sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'
           cleanWs()
           sh 'docker volume prune'
       }

          }
}
