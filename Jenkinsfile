pipeline {
    agent {
        label 'master'
    }
    stages {
        stage('Clean') {
            steps {
                sh 'rm -rf out'
            }
        }
        stage('Transform') {
            agent {
                docker {
                    image 'cloudfluff/databaker'
                    reuseNode true
                }
            }
            steps {
                sh "jupyter-nbconvert --to python --stdout 'Long-term international migration 2.01a tidydata.ipynb' | ipython"
            }
        }
        stage('RDF Data Cube') {
            agent {
                docker {
                    image 'cloudfluff/table2qb'
                    reuseNode true
                }
            }
            steps {
                sh "table2qb exec cube-pipeline --input-csv out/tidydata2_1.csv --output-file out/observations.ttl --column-config metadata/columns.csv --dataset-name 'ONS-LTIM-citizenship' --base-uri http://gss-data.org.uk/ --dataset-slug ons-ltim-citizenship"
            }
        }
        stage('Upload draftset') {
            steps {
                script {
                    String datasetLabel = 'ONS-LTIM-citizenship'
                    configFileProvider([configFile(fileId: 'pmd', variable: 'configfile')]) {
                        def config = readJSON(text: readFile(file: configfile))
                        String PMD = config['pmd_api']
                        String credentials = config['credentials']
                        String PIPELINE = config['pipeline_api']
                        def drafts = drafter.listDraftsets(PMD, credentials, 'owned')
                        def jobDraft = drafts.find { it['display-name'] == env.JOB_NAME }
                        if (jobDraft) {
                            drafter.deleteDraftset(PMD, credentials, jobDraft.id)
                        }
                        def newJobDraft = drafter.createDraftset(PMD, credentials, env.JOB_NAME)
                        String datasetPath = datasetLabel.toLowerCase()
                            .replaceAll('[^\\w/]', '-')
                            .replaceAll('-+', '-')
                            .replaceAll('-\$', '')
                        drafter.deleteGraph(PMD, credentials, newJobDraft.id,
                                            "http://gss-data.org.uk/graph/${datasetPath}/metadata")
                        drafter.deleteGraph(PMD, credentials, newJobDraft.id,
                                            "http://gss-data.org.uk/graph/${datasetPath}")
                        drafter.addData(PMD, credentials, newJobDraft.id,
                                        readFile("out/dataset.trig"), "application/trig;charset=UTF-8")
                        drafter.addData(PMD, credentials, newJobDraft.id,
                                        readFile("out/observations.ttl"), "text/turtle;charset=UTF-8",
                                        "http://gss-data.org.uk/graph/${datasetPath}")
                    }
                }
            }
        }
        stage('Publish') {
            steps {
                script {
                    publishDraftset()
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts 'out/*'
        }
    }
}
