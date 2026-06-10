// =============================================================================
//  Pipeline DevSecOps - TP Jenkins x OWASP Juice Shop
//  SCA (Trivy + npm audit) + SAST (Semgrep) + DAST (OWASP ZAP)
//  Déclenché automatiquement à chaque commit (webhook GitHub via ngrok),
//  avec pollSCM en filet de sécurité. Rapport envoyé par e-mail en post-build.
// =============================================================================
pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 40, unit: 'MINUTES')
    disableConcurrentBuilds()
  }

  // Déclenchement AUTOMATIQUE :
  //  - GitHub webhook -> coche "GitHub hook trigger for GITScm polling" (instantané)
  //  - pollSCM = filet de sécurité si le webhook tombe (vérifie toutes les 5 min)
  triggers {
    githubPush()
    pollSCM('H/5 * * * *')
  }

  environment {
    NET          = 'devsecops-net'
    JUICE_IMAGE  = 'bkimminich/juice-shop:latest'
    JUICE_NAME   = 'mon-juice-shop'                 // doit matcher l'URL de zap.yaml
    SEMGREP_IMG  = 'returntocorp/semgrep:latest'
    ZAP_IMG      = 'zaproxy/zap-stable:latest'
    TRIVY_IMG    = 'aquasec/trivy:latest'
  }

  stages {

    stage('Préparation') {
      steps {
        sh '''
          echo "=== Workspace : $WORKSPACE ==="
          docker network create ${NET} 2>/dev/null || echo "réseau ${NET} déjà présent"
          # On (pré)tire les images pour ne pas polluer la durée des stages
          docker pull ${SEMGREP_IMG} || true
          docker pull ${ZAP_IMG}     || true
          docker pull ${TRIVY_IMG}   || true
          docker pull ${JUICE_IMAGE} || true
        '''
      }
    }

    // -------------------------------------------------------------------------
    //  SCA : analyse des dépendances (Software Composition Analysis)
    // -------------------------------------------------------------------------
    stage('SCA - Dépendances (Trivy + npm audit)') {
      steps {
        sh '''
          echo "### Trivy : scan du système de fichiers (CVE des dépendances)"
          docker run --rm -v "$WORKSPACE":/src ${TRIVY_IMG} \
            fs --scanners vuln --severity HIGH,CRITICAL \
            --format json -o /src/sca_trivy.json /src || true

          echo "### Trivy : scan de l'image Juice Shop"
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ${TRIVY_IMG} \
            image --severity HIGH,CRITICAL ${JUICE_IMAGE} \
            | tee "$WORKSPACE/sca_trivy_image.txt" || true

          echo "### npm audit (génère un lockfile temporaire puis audite)"
          docker run --rm -v "$WORKSPACE":/src -w /src node:20-alpine sh -c \
            "npm install --package-lock-only --ignore-scripts >/dev/null 2>&1; npm audit --json" \
            > "$WORKSPACE/sca_npm_audit.json" 2>/dev/null || true
        '''
      }
    }

    // -------------------------------------------------------------------------
    //  SAST : analyse statique du code source (Semgrep)
    // -------------------------------------------------------------------------
    stage('SAST - Code source (Semgrep)') {
      steps {
        sh '''
          echo "### Semgrep (règles auto) -> SARIF + JSON + résumé texte"
          docker run --rm -v "$WORKSPACE":/src ${SEMGREP_IMG} \
            semgrep --config=auto --sarif -o /src/rapport.sarif /src || true

          docker run --rm -v "$WORKSPACE":/src ${SEMGREP_IMG} \
            semgrep --config=auto --json -o /src/rapport.json /src || true

          # Résumé lisible pour le corps du mail
          docker run --rm -v "$WORKSPACE":/src ${SEMGREP_IMG} \
            semgrep --config=auto /src 2>/dev/null \
            | tail -n 40 > "$WORKSPACE/sast_resume.txt" || true
        '''
      }
    }

    // -------------------------------------------------------------------------
    //  DAST : test dynamique de l'app en cours d'exécution (OWASP ZAP)
    // -------------------------------------------------------------------------
    stage('DAST - Application live (OWASP ZAP)') {
      steps {
        sh '''
          echo "### Démarrage de Juice Shop sur le réseau ${NET}"
          docker rm -f ${JUICE_NAME} 2>/dev/null || true
          docker run -d --name ${JUICE_NAME} --network ${NET} ${JUICE_IMAGE}

          echo "### Attente que l'application réponde (max ~120s)"
          for i in $(seq 1 40); do
            if docker run --rm --network ${NET} ${TRIVY_IMG} version >/dev/null 2>&1; then :; fi
            if docker run --rm --network ${NET} curlimages/curl:latest \
                 -s -o /dev/null -w "%{http_code}" http://${JUICE_NAME}:3000 | grep -q 200; then
              echo "Juice Shop est prêt."
              break
            fi
            echo "  ...en attente ($i)"; sleep 3
          done

          echo "### Lancement du scan ZAP (Automation Framework via zap.yaml)"
          docker run --rm --network ${NET} \
            -v "$WORKSPACE":/zap/wrk:rw \
            ${ZAP_IMG} zap.sh -cmd -autorun /zap/wrk/zap.yaml || true
        '''
      }
    }
  }

  // ---------------------------------------------------------------------------
  //  POST-BUILD : archivage + publication HTML + ENVOI DU RAPPORT PAR MAIL
  // ---------------------------------------------------------------------------
  post {
    always {
      // On nettoie le conteneur cible du DAST
      sh 'docker rm -f ${JUICE_NAME} 2>/dev/null || true'

      // Extrait SAST sûr pour le corps du mail (évite une erreur si le fichier manque)
      script {
        env.SAST_EXCERPT = fileExists('sast_resume.txt') ? readFile('sast_resume.txt').take(2000) : 'Aucun résumé SAST disponible.'
      }

      // Archivage des rapports comme artefacts du build
      archiveArtifacts artifacts: 'rapport_dast.html, rapport.sarif, rapport.json, sca_*.json, sca_*.txt, sast_resume.txt',
                       allowEmptyArchive: true, fingerprint: true

      // Publication du rapport ZAP dans l'UI Jenkins (plugin HTML Publisher)
      publishHTML(target: [
        reportName : 'Rapport DAST (ZAP)',
        reportDir  : '.',
        reportFiles: 'rapport_dast.html',
        keepAll    : true, alwaysLinkToLastBuild: true, allowMissing: true
      ])

      // Envoi du rapport de build par e-mail (plugin Email Extension)
      emailext(
        to:      'elimaneka@esp.sn',
        subject: "[Jenkins][TP DevSecOps] ${currentBuild.currentResult} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        mimeType: 'text/html',
        body: """
          <h2>Rapport de build DevSecOps</h2>
          <ul>
            <li><b>Job</b> : ${env.JOB_NAME} #${env.BUILD_NUMBER}</li>
            <li><b>Résultat</b> : ${currentBuild.currentResult}</li>
            <li><b>Commit</b> : ${env.GIT_COMMIT ?: 'n/a'}</li>
            <li><b>Durée</b> : ${currentBuild.durationString}</li>
            <li><b>Console</b> : <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></li>
          </ul>
          <p>En pièces jointes : rapport DAST (ZAP), SAST (Semgrep, SARIF/JSON) et SCA (Trivy / npm audit).</p>
          <hr><pre>Extrait SAST :\n${env.SAST_EXCERPT}</pre>
        """,
        attachmentsPattern: 'rapport_dast.html, rapport.sarif, sca_trivy.json, sca_npm_audit.json',
        attachLog: true
      )
    }
  }
}
