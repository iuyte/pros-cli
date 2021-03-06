stage('Build') {
  def build_ver = '2.4.2'
  parallel unix: {
    node('linux') {
      def venv = new edu.purdue.pros.venv()
      stage('Clean') {
        sh 'git init'
        sh 'git clean -d -x -f'
      }
      stage('Dependencies') {
        tool 'python3'
		if(isUnix()) {
	        sh 'sudo apt-get install -y python3-pip'
		}
        venv.create_virtualenv()
        venv.run 'pip3 install wheel twine'
      }
      stage('Clone') {
        checkout scm
        sh 'git describe --tags > version'
        build_ver = readFile 'version'
        build_ver = build_ver.replaceAll("\\s","")
        println "Building CLI at version ${build_ver}"
      }
      stage('Build') {
        venv.run 'python setup.py bdist_wheel'
        dir('dist') {
          archiveArtifacts artifacts: 'pros_cli-*-none-any.whl', fingerprint: true
        }
      }
    }
  }, windows64: {
    node('win&&x64') {
      def venv = new edu.purdue.pros.venv()
      stage('Clean') {
        bat "${tool name: 'Default', type: 'git'} init"
        bat "${tool name: 'Default', type: 'git'} clean -d -x -f"
      }
      stage('Dependenices') {
        tool 'MSBuild'
        venv.create_virtualenv()
        dir('cx_Freeze') {
          checkout changelog: false, poll: false, scm: [$class: 'MercurialSCM', credentialsId: '', installation: 'Mercurial', source: 'https://bitbucket.org/anthony_tuininga/cx_freeze']
          venv.run 'pip3 install .'
        }
      }
      stage('Clone') {
        checkout scm
        bat 'git describe --tags > version'
        build_ver = readFile 'version'
        build_ver = build_ver.replaceAll("\\s","")
        bat 'git describe --tags --abbrev=0 > inst_version'
        inst_ver = readFile 'inst_version'
        inst_ver = inst_ver.replaceAll("\\s","")
      }
      stage('Build') {
        venv.run 'pip3 install --upgrade -r requirements.txt'
        venv.run 'python build.py build_exe'
        archiveArtifacts artifacts: 'pros_cli-*-win*.zip', fingerprint: true
      }
    }
  }
  stage('Windows Installers') {
    node('win&&advinst') {
      ws {
        git credentialsId: 'phabricator-sigbot-ssh-key', poll: false, url: 'ssh://git@phabricator.purduesigbots.com/diffusion/WININSTALLER/pros-windows-installers.git'
        bat 'if exist .\\exe.win del /s /q .\\exe.win'
        bat 'if exist .\\exe.win rmdir /s /q .\\exe.win'
        bat 'mkdir .\\exe.win'
        for(file in unarchive(mapping: ['**pros_cli-*-win-32bit.zip': '.'])) {
          file.unzip(file.getParent().child('exe.win'))
        }
        def advinst = "\"${tool 'Advanced Installer'}\\AdvancedInstaller.com\""
        bat """
            ${advinst} /edit pros-windows.aip /SetVersion ${inst_ver}
            ${advinst} /edit pros-windows.aip /ResetSync APPDIR\\cli -clearcontent
            ${advinst} /edit pros-windows.aip /NewSync APPDIR\\cli exe.win -existingfiles delete
            ${advinst} /build pros-windows.aip
            """
        bat """
            ${advinst} /edit pros-updates.aip /NewUpdate output\\pros-win.exe -name "PROS${inst_ver}" -display_name "PROS ${build_ver}" -url "${env.BUILD_URL}artifact/output/pros-win.exe"
            ${advinst} /build pros-updates.aip
            """
        archiveArtifacts artifacts: 'output/*', fingerprint: true
      }
    }
  }
}
doDeploy()
