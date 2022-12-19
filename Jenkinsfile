pipeline {
    agent any
    environment {
          TOKEN = credentials('GH_TOKEN')
          CR_PAT = credentials('DockerCR-cred')
          CR_USR = credentials('dockerhub-user')
          repo_name = 'bh-corporate-functions/trivy-bh'
          image_name = 'fifaweb'
          tag = '1.0.1'
          Packages_lock = '.'
	  PATH = "/bin:/usr/bin:usr/local/bin"
    }
    stages {
        stage('Build') {
            steps {
		// Clean before build
		cleanWs()
                // We need to explicitly checkout from SCM here
                checkout scm 
                echo "Testing.. ${repo_name}/${image_name}:${tag}"
		sh 'ls -al'
		sh 'docker build -t "${repo_name}/${image_name}:${tag}" .'
            }
        }
        stage('Docker Image scan trivy') {
            steps {
                echo 'Downloading shared policies and template file from DevSecOps shared-workflows repo....'
		sh 'git clone https://${TOKEN}@github.com/BH-Corporate-Functions/shared-workflows.git'
		// updating vulnerability database for docker Image Scan
		sh 'trivy image --download-db-only'
		echo ' Generate html report for os, fs  and docker misconfig vuln'
                sh 'mkdir -p reports'
		// Scan for image
		sh 'trivy image  --format template --template "@./shared-workflows/html.tpl" --ignorefile "/shared-workflows/.trivyignore" -o "reports/${image_name}.html" "${repo_name}/${image_name}:${tag}"'
		// Scan for fs (Trivy will look for vulnerabilities based on lock files such as Gemfile.lock and package-lock.json)
		sh 'trivy fs  --format template --template "@./shared-workflows/html.tpl" --ignorefile "/shared-workflows/.trivyignore" -o "reports/Project_name2.html" ${Packages_lock}'
		// Scan for Dockerfile misconfig(custom policies)
		sh 'trivy conf --policy ./shared-workflows/policies --format template --template "@./shared-workflows/html.tpl" -o "reports/Project_name3.html" --ignorefile "/shared-workflows/.trivyignore" --namespaces user ./Dockerfile'
		// Append all reports to Report.html
		sh 'cat reports/Project_name3.html >> reports/Project_name2.html;cat reports/Project_name2.html >> reports/${image_name}.html; rm -rf reports/Project_name* '
		// publish html vuln report
		publishHTML target : [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: '${image_name}.html',
                    reportName: 'Trivy Scan',
                    reportTitles: 'Trivy Scan'
                ]				
		echo 'check for high & critical vuln'
		sh 'trivy image --no-progress --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed --ignorefile "/shared-workflows/.trivyignore" "${repo_name}/${image_name}:${tag}"'
		sh 'trivy fs --no-progress --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed --ignorefile "/shared-workflows/.trivyignore" "${Packages_lock}"'	
		sh 'trivy conf --no-progress --exit-code 1 --severity CRITICAL --policy ./shared-workflows/policies  --ignorefile "/shared-workflows/.trivyignore" --namespaces user ./Dockerfile'    
            }
        }			
        stage('upload to docker registry') {
            steps {
                echo 'Testing..'
		sh 'echo $CR_PAT | docker login ghcr.io -u $CR_USR --password-stdin'
	        sh 'docker push  "${repo_name}/${image_name}:${tag}"'
            }
        }
    }
}
