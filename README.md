# 🎓 Infrastructure Moodle sur AWS - Architecture Scalable

> Déploiement haute disponibilité de Moodle LMS sur AWS avec architecture multi-tiers, Auto Scaling et performance optimisée.

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange.svg)](https://aws.amazon.com/)
[![Moodle](https://img.shields.io/badge/Moodle-LMS-blue.svg)](https://moodle.org/)
[![Infrastructure](https://img.shields.io/badge/IaC-CloudFormation-green.svg)](https://aws.amazon.com/cloudformation/)

## 📋 Vue d'ensemble

Ce projet implémente une architecture de référence pour déployer **Moodle** (Learning Management System) sur AWS avec une infrastructure scalable, hautement disponible et sécurisée. Conçu pour supporter des charges variables d'étudiants et formateurs avec une performance optimale.

### 🎯 Cas d'usage
- Plateforme e-learning pour entreprises et institutions
- Support de milliers d'utilisateurs simultanés
- Déploiement multi-régions pour couverture internationale
- Intégration avec systèmes existants (ERP, CRM)

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Route 53 (DNS)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   CloudFront (CDN)                          │
│              Cache statique + SSL/TLS                       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              Application Load Balancer                      │
│           Distribution multi-AZ + Health checks             │
└─────────┬──────────────────────────────┬────────────────────┘
          │                              │
┌─────────▼─────────┐          ┌─────────▼─────────┐
│   Auto Scaling    │          │   Auto Scaling    │
│   EC2 - AZ1       │          │   EC2 - AZ2       │
│  PHP + Moodle     │          │  PHP + Moodle     │
└─────────┬─────────┘          └─────────┬─────────┘
          │                              │
          └──────────┬───────────────────┘
                     │
    ┌────────────────┼────────────────┬──────────────┐
    │                │                │              │
┌───▼────┐    ┌─────▼─────┐   ┌─────▼─────┐   ┌───▼────┐
│ RDS    │    │ElastiCache│   │    EFS    │   │   S3   │
│Aurora  │    │  Redis    │   │ Shared    │   │Storage │
│MySQL   │    │  Memcache │   │  Files    │   │ Backup │
└────────┘    └───────────┘   └───────────┘   └────────┘
```

## ✨ Fonctionnalités

### Infrastructure
- **Auto Scaling** : Ajustement automatique selon la charge (CPU, RAM, connexions)
- **Haute disponibilité** : Déploiement multi-AZ avec failover automatique
- **Performance** : ElastiCache (Redis + Memcached) pour sessions et cache
- **Stockage** : EFS pour fichiers partagés, S3 pour backups et médias
- **Sécurité** : VPC isolé, Security Groups, IAM roles, SSL/TLS obligatoire

### Base de données
- **Aurora MySQL** : Réplication multi-AZ, snapshots automatiques
- **Read Replicas** : Séparation lecture/écriture pour performance
- **Backup** : Rétention 30 jours, restauration point-in-time

### Monitoring & Logs
- **CloudWatch** : Métriques système et application en temps réel
- **Alarmes** : Notifications automatiques (CPU, mémoire, erreurs)
- **Logs** : Centralisation CloudWatch Logs (Apache, PHP, Moodle)

## 🛠️ Stack Technique

| Composant | Technologie | Rôle |
|-----------|-------------|------|
| **Compute** | EC2 t3.medium/large | Serveurs applicatifs Moodle |
| **Load Balancer** | Application LB | Distribution de charge |
| **Database** | Aurora MySQL 5.7 | Base de données principale |
| **Cache** | ElastiCache Redis | Sessions utilisateurs + cache objet |
| **Storage** | EFS + S3 | Fichiers partagés + backups |
| **CDN** | CloudFront | Distribution contenu statique |
| **Monitoring** | CloudWatch | Métriques et logs |
| **IaC** | CloudFormation | Infrastructure as Code |

## 🚀 Déploiement

### Prérequis
```bash
- Compte AWS avec permissions AdministratorAccess
- AWS CLI configuré
- Clé SSH pour accès EC2
- Domaine DNS (optionnel pour Route 53)
```

### Installation rapide

1. **Cloner le repository**
```bash
git clone https://github.com/elouafi-abderrahmane-2002/aws-refarch-moodle.git
cd aws-refarch-moodle
```

2. **Configurer les paramètres**
```bash
cp parameters.example.json parameters.json
# Éditer parameters.json avec vos valeurs
```

3. **Déployer la stack CloudFormation**
```bash
aws cloudformation create-stack \
  --stack-name moodle-production \
  --template-body file://cloudformation/main-stack.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_IAM
```

4. **Suivre le déploiement**
```bash
aws cloudformation describe-stacks \
  --stack-name moodle-production \
  --query 'Stacks[0].StackStatus'
```

### Configuration Moodle

Une fois l'infrastructure déployée :

```bash
# Récupérer l'URL du Load Balancer
ALB_DNS=$(aws cloudformation describe-stacks \
  --stack-name moodle-production \
  --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerDNS`].OutputValue' \
  --output text)

echo "Accéder à Moodle: http://$ALB_DNS/install.php"
```

## ⚙️ Configuration

### Paramètres CloudFormation

```yaml
Parameters:
  EnvironmentName: production
  InstanceType: t3.medium
  MinSize: 2          # Instances minimum
  MaxSize: 10         # Instances maximum
  DesiredCapacity: 2  # Instances initiales
  DBInstanceClass: db.t3.medium
  DBAllocatedStorage: 100
  MoodleVersion: "4.1"
  EnableCloudFront: true
  SSLCertificateArn: arn:aws:acm:...
```

### Variables d'environnement Moodle

```bash
# config.php généré automatiquement
CFG->dbtype    = 'mysqli';
CFG->dbhost    = '<RDS_ENDPOINT>';
CFG->dbname    = 'moodle';
CFG->dbuser    = 'moodleadmin';
CFG->wwwroot   = 'https://lms.example.com';
CFG->dataroot  = '/mnt/efs/moodledata';
CFG->sessioncache = 'redis';
CFG->cachestore_redis = '<ELASTICACHE_ENDPOINT>';
```

## 📊 Performance & Scaling

### Métriques Auto Scaling

| Métrique | Seuil Scale-Out | Seuil Scale-In |
|----------|----------------|----------------|
| CPU | > 70% pendant 5min | < 30% pendant 10min |
| Connexions ALB | > 1000 par instance | < 300 par instance |
| Mémoire | > 80% | < 40% |

### Optimisations

- **OpCache PHP** activé (128MB)
- **Redis** pour sessions et cache MUC
- **CloudFront** cache 24h pour assets statiques
- **EFS Provisioned Throughput** 100 MB/s
- **Aurora Read Replicas** pour requêtes SELECT

## 🔒 Sécurité

### Réseau
- VPC isolé avec subnets publics/privés
- Security Groups restrictifs (ports 80/443 uniquement)
- NAT Gateway pour accès internet EC2 privés
- Network ACLs pour filtrage supplémentaire

### Données
- Encryption at rest (EBS, RDS, S3)
- Encryption in transit (SSL/TLS)
- Backups automatiques chiffrés
- Secrets dans AWS Secrets Manager

### Accès
- IAM Roles pour EC2 (pas de credentials hardcodés)
- MFA obligatoire pour accès console AWS
- SSH désactivé (SSM Session Manager)

## 📈 Monitoring

### Dashboards CloudWatch

- **Application** : Temps de réponse, erreurs 5xx, utilisateurs actifs
- **Infrastructure** : CPU, RAM, disque, réseau
- **Database** : Connexions, queries/sec, replication lag
- **Cache** : Hit ratio, evictions, memory usage

### Alarmes critiques

```bash
- EC2 CPU > 90% pendant 10 minutes
- RDS Disk Space < 10%
- ALB Unhealthy Host Count > 0
- ElastiCache Memory > 90%
- Erreurs 5xx > 100 en 5 minutes
```

## 💰 Estimation des coûts

**Configuration moyenne** (Europe - Paris)

| Service | Configuration | Coût mensuel |
|---------|--------------|--------------|
| EC2 (2x t3.medium) | On-Demand | ~€60 |
| RDS Aurora (db.t3.medium) | Multi-AZ | ~€120 |
| ElastiCache Redis | cache.t3.micro | ~€20 |
| EFS | 500 GB | ~€150 |
| S3 | 1 TB + requêtes | ~€25 |
| ALB | Inclus transfert | ~€25 |
| CloudFront | 1 TB transfert | ~€85 |
| **TOTAL** | | **~€485/mois** |

*Note: Coûts variables selon utilisation réelle*

## 🔄 Maintenance

### Backups

```bash
# RDS snapshots automatiques quotidiens (rétention 30j)
# Scripts de backup EFS vers S3 (cron quotidien)
aws backup start-backup-job --backup-vault-name moodle-vault
```

### Mises à jour Moodle

```bash
# Rolling update sans downtime
./scripts/update-moodle.sh --version 4.2 --strategy rolling
```

### Scaling manuel

```bash
# Augmenter temporairement la capacité
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name moodle-asg \
  --desired-capacity 5
```

## 🎓 Cas d'usage réels

- **Quiz Coach** : Support de 10,000+ étudiants simultanés
- **Université** : 50,000 utilisateurs, 500 cours actifs
- **Formation entreprise** : Intégration SSO SAML avec AD

## 📚 Documentation

- [Architecture Decision Records](docs/architecture/)
- [Runbook opérationnel](docs/operations/)
- [Guide troubleshooting](docs/troubleshooting.md)
- [API Moodle integration](docs/api-integration.md)

## 🤝 Contribution

Les contributions sont bienvenues ! 

1. Fork le projet
2. Créer une branche (`git checkout -b feature/amelioration`)
3. Commit (`git commit -m 'Add: nouvelle fonctionnalité'`)
4. Push (`git push origin feature/amelioration`)
5. Ouvrir une Pull Request

## 📝 Licence

MIT License - voir [LICENSE](LICENSE) pour détails

## 👤 Auteur

**Abderrahmane ELOUAFI**
- GitHub: [@elouafi-abderrahmane-2002](https://github.com/elouafi-abderrahmane-2002)
- LinkedIn: [abderrahmane-elouafi](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/)
- Email: elouafi.abderrahmane.work@gmail.com

---

⭐ **Star ce repo** si vous le trouvez utile !
