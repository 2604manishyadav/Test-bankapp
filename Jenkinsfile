pipeline {
    agent any

    parameters {
        string(name: 'Image_version', defaultValue: 'latest', description: 'Image Tag')
    }

    stages {
        stage("Clone Code") {
            steps {
                git url: "https://github.com/2604manishyadav/Bankapp.git", branch: "mega-project"
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t bankapp-eks:${params.Image_version} ."
            }
        }

        stage("Push to Docker Hub") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerHub',
                    usernameVariable: 'DockerHubUser',
                    passwordVariable: 'DockerHubPass'
                )]) {
                    sh "docker login -u $DockerHubUser -p $DockerHubPass"
                    sh "docker image tag bankapp-eks:${params.Image_version} $DockerHubUser/bankapp-eks:${params.Image_version}"
                    sh "docker push $DockerHubUser/bankapp-eks:${params.Image_version}"
                }
            }
        }
    }
}
