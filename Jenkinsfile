def createPackages(String architecture, String version) {
    script {
        def fpmArchitecture = architecture == 'arm64' ? 'aarch64' : architecture
        sh """
        mkdir -p build/usr/bin
        mv build/orchent-${architecture}-linux build/usr/bin/orchent
        fpm -s dir -t deb -a ${fpmArchitecture} -n orchent -v ${version} -C build/ \\
            -p orchent_${version}_${architecture}.deb .
        fpm -s dir -t rpm -a ${fpmArchitecture} -n orchent -v ${version} -C build/ \\
            -p orchent_${version}_${architecture}.rpm .    
        """
    }
}

def getReleaseVersion(String tagName) {
    if (tagName) {
        return tagName.replaceAll(/^v/, '')
    } else {
        return null
    }
}

pipeline {
  agent none
  environment {
        // Set RELEASE_VERSION only if TAG_NAME is set
        RELEASE_VERSION = getReleaseVersion(TAG_NAME)
    }
  stages {
        stage('Build') {
            agent {
                docker {
                  label 'jenkins-node-label-1'
                  image 'golang:1.16.15'
                  reuseNode true
                }
            }
            steps {
                script {
                  sh '''
                  mkdir -p build
                  export GOCACHE=$WORKSPACE/.cache/go-build
                  CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -ldflags ' -w -extldflags "-static"' -ldflags "-B 0x$(head -c20 /dev/urandom|od -An -tx1|tr -d ' \n')" -o build/orchent-amd64-linux orchent.go
                  CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -a -ldflags ' -w -extldflags "-static"' -ldflags "-B 0x$(head -c20 /dev/urandom|od -An -tx1|tr -d ' \n')" -o build/orchent-arm64-linux orchent.go
                  CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build -a -ldflags '-w -extldflags "-static"' -o build/orchent-macos-amd64 orchent.go
                  CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build -a -ldflags '-w -extldflags "-static"' -o build/orchent-macos-arm64 orchent.go
                  '''
                }  
            }
        }
        
        stage('Package') {
            when { tag "v*"}
            agent {
                docker {
                    label 'jenkins-node-label-1'
                    image 'marica/fpm:latest'
                    reuseNode true
                }
            }
            options { skipDefaultCheckout() }
            steps {
                    createPackages('amd64', RELEASE_VERSION)
                    createPackages('arm64', RELEASE_VERSION)
            }
        }
        stage('Upload to Nexus'){
          when { tag "v*"}
          agent {
                node { label 'jenkins-node-label-1' }
            }
          options { skipDefaultCheckout() }
          steps{
            nexusArtifactUploader(
              nexusVersion: 'nexus3',
              protocol: 'https',
              nexusUrl: 'repo.cloud.cnaf.infn.it',
              version: RELEASE_VERSION,
              repository: 'orchent',
              groupId: '',
              credentialsId: 'nexus-credentials',
              artifacts: [ 
                  [ artifactId: 'orchent-amd64', type: 'deb', classifier: '', file: "orchent_${RELEASE_VERSION}_amd64.deb" ],
                  [ artifactId: 'orchent-arm64', type: 'deb', classifier: '', file: "orchent_${RELEASE_VERSION}_arm64.deb" ],
                  [ artifactId: 'orchent-amd64', type: 'rpm', classifier: '', file: "orchent_${RELEASE_VERSION}_amd64.rpm" ],
                  [ artifactId: 'orchent-arm64', type: 'rpm', classifier: '', file: "orchent_${RELEASE_VERSION}_arm64.rpm" ],
                  [ artifactId: 'orchent-arm64', type: 'rpm', classifier: '', file: "orchent_${RELEASE_VERSION}_arm64.rpm" ],
                  [ artifactId: 'orchent-arm64', type: '', classifier: 'macos', file: "build/orchent-macos-arm64" ],
                  [ artifactId: 'orchent-amd64', type: '', classifier: 'macos', file: "build/orchent-macos-amd64" ]
              ]
            )
             
            }
            post {
               always {
                 cleanWs()
               }
             }
          }
  }
}
