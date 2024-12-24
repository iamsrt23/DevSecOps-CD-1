pipeline{
    agent {label 'build'}
    parameters{
        password(name: 'PASSWD', defaultValue: '', description: 'Please Enter Your GitHub Password')
        string(name: 'IMAGETAG', defaultValue: '1',description: 'Please Enter Image Tag to Deploy' )
    }
    stages{
        stage('Deploy'){
            steps{
                git branch: 'main', credentialsId: 'githubtoken', url:'https://github.com/iamsrt23/DevSecOps-CD-1.git'
                dir("./kubernetes"){
                    sh "sed -i 's/image: iamsrt23.*/image: iamsrt23\\/democicd:$IMAGETAG/g' deployment.yaml"

                }
                sh 'git commit -a -m "New Deployment for Build $IMAGETAG'
                sh "git push https://iamsrt23:$PASSWD@gihub.com/iamsrt23/DevSecOps-CD-1.git"

            }
        }
    }
}