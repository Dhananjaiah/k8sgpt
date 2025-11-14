# K8sGPT Course + SRE Runbook

Welcome to the complete, hands-on **K8sGPT Course and Runbook** for real-world Kubernetes cluster troubleshooting!

## 🎯 What You'll Learn

This course teaches you how to use **K8sGPT** - an AI-powered SRE assistant that helps you diagnose and troubleshoot Kubernetes cluster issues faster and more effectively than traditional tools.

By the end of this course, you will:

- ✅ Understand what K8sGPT is and when to use it
- ✅ Set up a local Kubernetes lab environment
- ✅ Install and configure K8sGPT CLI
- ✅ Run scans to detect cluster issues automatically
- ✅ Interpret K8sGPT output and apply fixes
- ✅ Automate K8sGPT in CI/CD pipelines
- ✅ Use K8sGPT effectively during production incidents
- ✅ Follow SRE best practices for cluster troubleshooting

## 👥 Who This Course Is For

This course is designed for:

- **DevOps Engineers** who manage Kubernetes clusters
- **Site Reliability Engineers (SREs)** who respond to incidents
- **Platform Engineers** building internal Kubernetes platforms
- **System Administrators** transitioning to Kubernetes
- **Developers** who want to understand their deployments better

## 📋 Prerequisites

Before starting this course, you should have:

- **Basic Kubernetes knowledge**: Understanding of Pods, Deployments, Services, ConfigMaps, Secrets
- **kubectl installed**: Command-line tool for Kubernetes
- **A test cluster**: kind, minikube, or any lab/development cluster (we'll help you set this up)
- **kubectl context configured**: Ability to run `kubectl get nodes`
- **Basic command-line skills**: Comfortable with terminal/bash commands

**Optional but helpful:**
- Docker or Podman installed
- Basic understanding of YAML
- Experience with `kubectl describe` and `kubectl logs`

## 📚 Course Structure

Follow the course in order for the best learning experience:

### Part 1: Understanding K8sGPT

1. **[Introduction to K8sGPT](docs/01-intro-k8sgpt.md)**
   - What is K8sGPT and why use it?
   - Core features and capabilities
   - How it compares to traditional tools

### Part 2: Setup and Configuration

2. **[Lab Setup](docs/02-lab-setup.md)**
   - Create a local Kubernetes cluster
   - Install required tools
   - Verify your environment

3. **[Install K8sGPT CLI](docs/03-install-k8sgpt-cli.md)**
   - Install K8sGPT on your system
   - Configure authentication
   - Verify installation

4. **[Connect K8sGPT to Your Cluster](docs/04-connect-k8sgpt-to-cluster.md)**
   - Configure kubeconfig
   - Set up cluster context
   - Authenticate with LLM backend

### Part 3: Using K8sGPT

5. **[Basic Scans](docs/05-k8sgpt-basic-scans.md)**
   - Run your first scan
   - Understand the output
   - Filter and scope scans

6. **[Demo: Broken Application](docs/06-k8sgpt-demo-broken-app.md)**
   - Deploy a broken application
   - Use K8sGPT to find issues
   - Apply fixes and verify

7. **[Advanced Usage and Analyzers](docs/07-advanced-usage-analyzers.md)**
   - Deep dive into analyzers
   - Advanced filtering options
   - Configuration management

### Part 4: Production Usage

8. **[Integrations and Automation](docs/08-integrations-and-automation.md)**
   - Automate scans with CronJobs
   - CI/CD pipeline integration
   - Connect to monitoring tools

9. **[Security and Cost Considerations](docs/09-security-and-cost-considerations.md)**
   - Protect API keys and secrets
   - Network security
   - Cost management strategies

### Part 5: Operational Guides

10. **[SRE Runbook](docs/10-k8sgpt-sre-runbook.md)**
    - Complete operational guide
    - Common incident scenarios
    - Step-by-step troubleshooting

11. **[Troubleshooting K8sGPT](docs/11-troubleshooting-k8sgpt-itself.md)**
    - Fix K8sGPT issues
    - Common problems and solutions
    - Debugging techniques

## 🚀 Quick Start

If you're familiar with Kubernetes and want to jump right in:

```bash
# 1. Create a cluster (if you don't have one)
kind create cluster --name k8sgpt-demo

# 2. Install K8sGPT
# On macOS/Linux:
brew install k8sgpt

# On Linux (alternative):
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz
tar -xvf k8sgpt_Linux_x86_64.tar.gz
sudo mv k8sgpt /usr/local/bin/

# 3. Configure authentication (optional - can use anonymous mode)
k8sgpt auth add --backend openai --password YOUR_OPENAI_API_KEY

# 4. Run your first scan
k8sgpt analyze --explain

# 5. Deploy a broken app and scan it
kubectl apply -f k8s/broken-app.yaml
k8sgpt analyze --namespace default --explain
```

## 📖 Operational Runbooks

For on-call engineers and SREs:

- **[Quick Runbook](RUNBOOK.md)** - Fast reference for incidents
- **[Full SRE Runbook](docs/10-k8sgpt-sre-runbook.md)** - Comprehensive guide

## 🛠️ Example Manifests

The `k8s/` directory contains ready-to-use examples:

- **[broken-app.yaml](k8s/broken-app.yaml)** - Demo application with common issues
- **[cronjob-k8sgpt-scan.yaml](k8s/cronjob-k8sgpt-scan.yaml)** - Automated scanning setup

## 🎓 Learning Path

**Beginner (1-2 hours):**
- Complete modules 1-6
- Deploy and fix the broken app
- Understand basic K8sGPT commands

**Intermediate (3-4 hours):**
- Complete all modules
- Set up automated scanning
- Practice with your own applications

**Advanced (1 week):**
- Integrate K8sGPT into your CI/CD
- Customize analyzers for your environment
- Build incident response workflows

## 🤝 Contributing

Found an issue or want to improve the course? Contributions are welcome!

## 📝 License

This course material is provided as-is for educational purposes.

## 🆘 Getting Help

- Review the [Troubleshooting Guide](docs/11-troubleshooting-k8sgpt-itself.md)
- Check the [K8sGPT documentation](https://docs.k8sgpt.ai/)
- Open an issue in this repository

## 🚦 Getting Started

Ready to begin? Start with **[Introduction to K8sGPT](docs/01-intro-k8sgpt.md)** →

---

**Remember:** This course uses a safe lab environment. Never run untested commands in production!
