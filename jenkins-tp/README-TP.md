# TP DevSecOps — Jenkins × OWASP Juice Shop (SAST / DAST / SCA)

Guide de mise en place complet. Chaque section correspond à une partie attendue du rapport.

- **Projet vulnérable** : OWASP Juice Shop (fork GitHub personnel)
- **SAST** : Semgrep → `rapport.sarif` / `rapport.json`
- **DAST** : OWASP ZAP (Automation Framework, `zap.yaml`) → `rapport_dast.html`
- **SCA** : Trivy (fs + image) + `npm audit`
- **Orchestration** : Jenkins (Docker) + `Jenkinsfile` (Pipeline as Code)
- **Déclenchement auto** : webhook GitHub via ngrok (+ pollSCM en secours)
- **Notification** : plugin Email Extension (`emailext`)

---

## Architecture

```
   git push (commit)
        │
        ▼
   GitHub (fork juice-shop) ──webhook──► ngrok ──► Jenkins (Docker, :8080)
                                                        │  utilise le socket Docker de l'hôte
                                                        ▼
                         ┌──────────────┬──────────────┬───────────────┐
                         ▼              ▼              ▼               ▼
                     Trivy/npm       Semgrep          ZAP  ───►  mon-juice-shop
                       (SCA)          (SAST)         (DAST)      (réseau devsecops-net)
                         │              │              │
                         └──────────────┴──────────────┴──► rapports ──► e-mail (emailext)
```

---

## Partie A — Manips ZAP + Juice Shop (DAST manuel) ✅ déjà réalisé

Ces commandes correspondent à `zap.yaml` + `rapport_dast.html` déjà présents.

```bash
# 1. Lancer la cible
docker network create devsecops-net
docker run -d --name mon-juice-shop --network devsecops-net bkimminich/juice-shop

# 2. Scan ZAP piloté par le plan d'automatisation zap.yaml
docker run --rm --network devsecops-net -v "$PWD":/zap/wrk:rw \
  zaproxy/zap-stable zap.sh -cmd -autorun /zap/wrk/zap.yaml

# 3. Résultat : rapport_dast.html
```

> Le fichier `.zap/rules.tsv` règle le bruit (IGNORE/OUTOFSCOPE) ; `zap.yaml` enchaîne
> `spider` → `passiveScan` → `report` (template traditional-html).

**À mettre dans le rapport** : captures du scan ZAP en cours, extraits de `rapport_dast.html`
(alertes XSS, SQLi, en-têtes manquants…), et le contenu commenté de `zap.yaml`.

---

## Partie B — Manips MANUELLES avec Jenkins (script + rapport de build)

### B.1 Lancer Jenkins (Docker)

```bash
cd jenkins-tp
docker compose up -d --build
docker logs jenkins 2>&1 | grep -A2 "Please use the following password"   # mot de passe initial
```

Ouvrir http://localhost:8080 → coller le mot de passe → **Install suggested plugins** →
créer l'utilisateur admin.

> Les plugins essentiels (`git`, `github`, `email-ext`, `workflow-aggregator`, `htmlpublisher`)
> sont déjà pré-installés par le `Dockerfile`.

### B.2 Premier build MANUEL (preuve que le pipeline fonctionne)

1. **New Item** → nom `juiceshop-devsecops` → **Pipeline** → OK.
2. Section *Pipeline* → **Pipeline script from SCM** → Git →
   URL = `https://github.com/<VOTRE_USER>/juice-shop.git`, branche `master`,
   *Script Path* = `Jenkinsfile`.
3. **Save** → **Build Now**.

**À mettre dans le rapport** : le `Jenkinsfile` commenté, la *Stage View* (SCA/SAST/DAST verts),
la console du build, et le mail reçu = « rapport de build ».

### B.3 Configurer l'e-mail (plugin Email Extension)

**Manage Jenkins → System → Extended E-mail Notification** :

| Champ | Valeur (exemple Gmail) |
|---|---|
| SMTP server | `smtp.gmail.com` |
| SMTP Port | `465` |
| Use SSL | ✔ |
| Credentials | *Add* → votre adresse + **mot de passe d'application** Gmail |
| Default Recipients | `elimaneka@esp.sn` |

> Gmail exige un **App Password** (compte avec 2FA), pas le mot de passe normal.
> Testez via le bouton *Test configuration by sending test e-mail*.

---

## Partie C — Déclenchement AUTOMATIQUE par commit (webhook GitHub + ngrok)

### C.1 Forker Juice Shop et pousser le pipeline

```bash
# Fork via l'UI GitHub : github.com/juice-shop/juice-shop -> Fork
# Puis sur votre machine, pointez 'origin' vers VOTRE fork :
git remote set-url origin https://github.com/<VOTRE_USER>/juice-shop.git

# Versionnez les artefacts du TP
git add Jenkinsfile zap.yaml .zap/rules.tsv jenkins-tp/
git commit -m "TP DevSecOps: pipeline Jenkins SAST/DAST/SCA"
git push origin master
```

### C.2 Exposer Jenkins avec ngrok

```bash
# Installer ngrok depuis l'AUR (Arch Linux) :
yay -S ngrok-v3-bin          # paquet AUR officiel ngrok v3
# Créer un compte gratuit sur ngrok.com pour récupérer votre authtoken
ngrok config add-authtoken <VOTRE_TOKEN>
ngrok http 8080
# -> note l'URL publique, ex: https://ab12cd34.ngrok-free.app
```

### C.3 Brancher le webhook GitHub

Sur **votre fork** → *Settings → Webhooks → Add webhook* :
- **Payload URL** : `https://ab12cd34.ngrok-free.app/github-webhook/`  (slash final obligatoire)
- **Content type** : `application/json`
- **Event** : *Just the push event* → **Add webhook** (doit afficher une coche verte ✅)

Côté job Jenkins → **Configure → Build Triggers** → cocher
**« GitHub hook trigger for GITScm polling »** (déjà géré par `triggers { githubPush() }` dans le Jenkinsfile).

### C.4 Démonstration « pas de build manuel »

```bash
# Un simple commit doit déclencher le pipeline tout seul
echo "// trigger $(date)" >> server.ts
git commit -am "test: déclenchement automatique du pipeline"
git push origin master
```

→ Jenkins démarre un build **automatiquement** (visible dans *Build History*, cause :
« Started by GitHub push »), exécute SCA/SAST/DAST, puis **envoie le rapport par mail**.

**À mettre dans le rapport** : capture du webhook GitHub (livraison 200 OK), du build déclenché
automatiquement (cause = GitHub push), et du mail de rapport reçu.

---

## Récapitulatif des livrables

| Exigence du TP | Preuve |
|---|---|
| Projet vulnérable + SAST/DAST/SCA auto | `Jenkinsfile` (stages SCA/SAST/DAST) |
| SAST | `rapport.sarif`, `rapport.json` (Semgrep) |
| DAST | `rapport_dast.html`, `zap.yaml` (ZAP) |
| SCA | `sca_trivy.json`, `sca_npm_audit.json`, `sca_trivy_image.txt` |
| Commit déclenche les tests (no build manuel) | Webhook GitHub + `triggers{ githubPush() }` |
| Rapport reçu par mail | `emailext` (post-build) |

## Dépannage rapide

- **`docker: permission denied` dans Jenkins** : vérifier `group_add: ["953"]` (= GID du groupe
  docker hôte, cf. `getent group docker`).
- **`docker run -v $WORKSPACE` monte un dossier vide** : le bind-mount `/var/jenkins_home`
  doit être **identique** dans/hors conteneur (déjà fait dans `docker-compose.yml`).
- **ZAP ne joint pas l'app** : conteneur cible nommé `mon-juice-shop` sur le réseau
  `devsecops-net` (l'URL dans `zap.yaml` doit correspondre).
- **Webhook ne déclenche pas** : URL ngrok à jour (elle change à chaque redémarrage de ngrok),
  slash final `/github-webhook/`, et trigger coché dans le job.
