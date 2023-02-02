#!/usr/bin/groovy

@Library(['github.com/indigo-dc/jenkins-pipeline-library@1.2.3']) _

pipeline {
    agent {
        label 'docker-build'
    }

    environment {
        dockerhub_repo = "deephdc/uc-hereariim-deep-oc-blossom"
        base_cpu_tag = "1.14.0-py3"
        base_gpu_tag = "1.14.0-gpu-py3"
    }

    stages {
        stage('Validate metadata') {
            steps {
                checkout scm
                sh 'deep-app-schema-validator metadata.json'
            }
        }
        stage('Docker image building') {
            when {
                anyOf {
                    branch 'master'
                    branch 'test'
                    buildingTag()
                }
            }
            steps{
                dir('check_oc_artifact'){
                    // clone checking scripts
                    git url: 'https://github.com/deephdc/deep-check_oc_artifact'
                }
                dir('deep-oc-user_app'){
                    checkout scm
                    script {
                        // build different tags
                        id = "${env.dockerhub_repo}"

                        if (env.BRANCH_NAME == 'master') {
                           // CPU (aka latest, i.e. default)
                           id_cpu = DockerBuild(id,
                                            tag: ['latest', 'cpu'],
                                            build_args: ["tag=${env.base_cpu_tag}",
                                                         "branch=master"])

                           // Check that the image starts and get_metadata responses correctly
                           sh "bash ../check_oc_artifact/check_artifact.sh ${env.dockerhub_repo}"

                           // GPU
                           id_gpu = DockerBuild(id,
                                            tag: ['gpu'],
                                            build_args: ["tag=${env.base_gpu_tag}",
                                                         "branch=master"])
                        }

                        if (env.BRANCH_NAME == 'test') {
                           // CPU
                           id_cpu = DockerBuild(id,
                                            tag: ['test', 'cpu-test'],
                                            build_args: ["tag=${env.base_cpu_tag}",
                                                         "branch=test"])

                           // Check that the image starts and get_metadata responses correctly
                           sh "bash ../check_oc_artifact/check_artifact.sh ${env.dockerhub_repo}:test"

                           // GPU
                           id_gpu = DockerBuild(id,
                                            tag: ['gpu-test'],
                                            build_args: ["tag=${env.base_gpu_tag}",
                                                         "branch=test"])
                        }
                    }
                }
            }
            post {
                failure {
                    DockerClean()
                }
            }
        }


        stage('Docker Hub delivery') {
            when {
                anyOf {
                   branch 'master'
                   branch 'test'
                   buildingTag()
               }
            }
            steps{
                script {
                    DockerPush(id_cpu)
                    DockerPush(id_gpu)
                }
            }
            post {
                failure {
                    DockerClean()
                }
                always {
                    cleanWs()
                }
            }
        }

        stage("Re-build DEEP-OC Docker images for derived services") {
            when {
                anyOf {
                branch 'master'
                branch 'test'
                buildingTag()
                }
            }
            steps {

                // Wait for the base image to be correctly updated in DockerHub as it is going to be used as base for
                // building the derived images
                sleep(time:5, unit:"MINUTES")

                script {
                    def derived_job_locations =
                    ['Pipeline-as-code/DEEP-OC-org/UC-hereariim-DEEP-OC-blossom'
                    ]

                    for (job_loc in derived_job_locations) {
                        job_to_build = "${job_loc}/${env.BRANCH_NAME}"
                        def job_result = JenkinsBuildJob(job_to_build)
                        job_result_url = job_result.absoluteUrl
                    }
                }
            }
        }

        stage("Render metadata on the marketplace") {
            when {
                allOf {
                    branch 'master'
                    changeset 'metadata.json'
                }
            }
            steps {
                script {
                    def job_result = JenkinsBuildJob("Pipeline-as-code/deephdc.github.io/pelican")
                    job_result_url = job_result.absoluteUrl
                }
            }
        }
    }
}
