#!/usr/bin/groovy

// function to remove built images
def docker_clean() {
    def dangling_images = sh(
		returnStdout: true,
		script: "docker images -f 'dangling=true' -q"
	)
    if (dangling_images) {
        sh(script: "docker rmi --force $dangling_images")
    }
}


pipeline {
    agent {
        label 'docker'
    }
    environment {
        // Remove .git from the GIT_URL link and extract REPO_NAME from GIT_URL
        REPO_URL = "${env.GIT_URL.endsWith(".git") ? env.GIT_URL[0..-5] : env.GIT_URL}"
        AI4OS_CVAT_REPO = "${REPO_URL.tokenize('/')[-1]}"
        DEFAULT_BRANCH = "v2.7.3-AI4OS"
        METADATA_VERSION = "2.0.0"
        AI4OS_REGISTRY_CREDENTIALS = credentials('AIOS-registry-credentials')
    }
    stages {
        stage("Variable initialization") {
            steps {
                script {
                    checkout scm
                    withFolderProperties{
                        env.DOCKER_REGISTRY = env.AI4OS_REGISTRY
                        env.DOCKER_REGISTRY_ORG = env.AI4OS_REGISTRY_REPOSITORY
                        env.DOCKER_REGISTRY_CREDENTIALS = env.AI4OS_REGISTRY_CREDENTIALS
                    }
                    // docker repository
                    env.DOCKER_REPO = env.DOCKER_REGISTRY_ORG + "/" + env.AI4OS_CVAT_REPO
                    // define base tag from branch name
                    env.IMAGE_TAG = "${env.BRANCH_NAME == 'main' ? 'latest' : env.BRANCH_NAME}"

                }
            }
        }

        stage('AI4OS Hub metadata V2 validation (YAML)') {
            when {
                // Check if ai4-metadata.yml is present in the repository
                expression {fileExists("ai4-metadata.yml")}
            }
            agent {
                docker {
                    image 'ai4oshub/ci-images:python3.12'
                }
            }
            steps {
                script {
                    if (!fileExists("ai4-metadata.yml")) {
                        error("ai4-metadata.yml file not found in the repository")
                    }
                    if (fileExists("ai4-metadata.json")) {
                        error("Both ai4-metadata.json and ai4-metadata.yml files found in the repository")
                    }
                }
                script {
                    sh "ai4-metadata validate --metadata-version ${env.METADATA_VERSION} ai4-metadata.yml"
                }
            }
        }
        stage("License validation") {
            steps {
                script {
                    // Check if LICENSE file is present in the repository
                    if (!fileExists("LICENSE")) {
                        error("LICENSE file not found in the repository")
                    }
                }
            }
        }

        stage("Check if only metadata files have changed") {
            steps {
                script {
                    // Check if only metadata files have been changed
                    // See https://github.com/ai4os/ai4os-hub-qa/issues/16
                    // If GIT_PREVIOUS_SUCCESSFUL_COMMIT fails
                    // (e.g. First time build, commits were rewritten by user),
                    // we fallback to last commit
                    try {
                        changed_files = sh (returnStdout: true, script: "git diff --name-only HEAD ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}").trim()
                    } catch (err) {
                        println("[WARNING] Exception: ${err}")
                        println("[INFO] Considering changes only in the last commit..")
                        changed_files = sh (returnStdout: true, script: "git diff --name-only HEAD^ HEAD").trim()
                    }
                    need_build = true

                    // Check if metadata files are present in the list of changed files
                    if (changed_files.contains("ai4-metadata.yml")) {
                        // Convert to an array and pop items
                        changed_files = changed_files.tokenize()
                        changed_files.removeAll(["metadata.json", "ai4-metadata.json", "ai4-metadata.yml"])
                        // now check if the list is empty
                        if (changed_files.size() == 0) {
                            need_build = false
                        }
                    }
                }
            }
        }

        stage("Docker build and push (Server)") {
            when {
                anyOf {
                    branch 'main'
                    branch env.DEFAULT_BRANCH
                    branch 'release/*'
                    buildingTag()
                }
                anyOf {
                    expression {need_build}
                    triggeredBy 'UserIdCause'
                }
            }
            steps {
                script {
                    checkout scm
                    dockerfile = "Dockerfile"
                    image_tag = env.IMAGE_TAG + "-server"
                    image = (env.DOCKER_REPO + ":" + image_tag).toLowerCase()
                    println ("[DEBUG] Config for the Docker image build: ${image}, push to $env.DOCKER_REGISTRY")
                    docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_REGISTRY_CREDENTIALS){
                         def app_image = docker.build(image,
                                                      "--no-cache --force-rm -f ${dockerfile} .")
                         app_image.push()
                    }
                }
            }
            post {
                failure {
                    docker_clean()
                }
            }
        }

        stage("Docker build and push (UI)") {
            when {
                anyOf {
                    branch 'main'
                    branch env.DEFAULT_BRANCH
                    branch 'release/*'
                    buildingTag()
                }
                anyOf {
                    expression {need_build}
                    triggeredBy 'UserIdCause'
                }
            }
            steps {
                script {
                    checkout scm
                    dockerfile = "Dockerfile.ui"
                    image_tag = env.IMAGE_TAG + "-ui"
                    image = (env.DOCKER_REPO + ":" + image_tag).toLowerCase()
                    println ("[DEBUG] Config for the Docker image build: ${image}, push to $env.DOCKER_REGISTRY")
                    docker.withRegistry(env.DOCKER_REGISTRY, env.DOCKER_REGISTRY_CREDENTIALS){
                         def app_image = docker.build(image,
                                                      "--no-cache --force-rm -f ${dockerfile} .")
                         app_image.push()
                    }
                }
            }
            post {
                failure {
                    docker_clean()
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
