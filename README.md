# n8n Local Kubernetes Stack 🚀

![n8n](https://img.shields.io/badge/n8n-automation-blue?style=flat-square)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-orange?style=flat-square)
![Kubernetes](https://img.shields.io/badge/Kubernetes-local-green?style=flat-square)
![Helm](https://img.shields.io/badge/Helm-chart-purple?style=flat-square)

Welcome to the **n8n Local Kubernetes Stack** repository! This project provides a production-ready setup for n8n, PostgreSQL 17, and Valkey, tailored for local Kubernetes environments. Our stack supports RWX NFS, Helm, queue mode, and worker/runner scaling. 

You can find the latest releases [here](https://github.com/devananda6/n8n-local-k8s-stack/releases). Please download and execute the necessary files to get started.

## Table of Contents

1. [Introduction](#introduction)
2. [Features](#features)
3. [Getting Started](#getting-started)
   - [Prerequisites](#prerequisites)
   - [Installation](#installation)
4. [Configuration](#configuration)
5. [Usage](#usage)
6. [Scaling](#scaling)
7. [Troubleshooting](#troubleshooting)
8. [Contributing](#contributing)
9. [License](#license)

## Introduction

n8n is an open-source workflow automation tool that enables users to connect various services and automate tasks. By combining it with PostgreSQL 17 and Valkey, you can build a powerful automation system in your local Kubernetes cluster. This setup ensures high availability and efficient resource management, making it ideal for production environments.

## Features

- **Automation**: Seamlessly connect multiple services and automate workflows.
- **DevOps Ready**: Built with DevOps principles in mind, ensuring easy deployment and management.
- **Helm Charts**: Simplify installation and updates using Helm.
- **Infrastructure as Code**: Manage your setup using YAML files for better version control.
- **Kubernetes**: Leverage Kubernetes for orchestration and scaling.
- **PostgreSQL 17**: Use the latest version of PostgreSQL for robust data management.
- **NFS Support**: RWX NFS for persistent storage.
- **Queue Mode**: Efficiently handle tasks with a queue system.
- **Worker/Runner Scaling**: Easily scale your workers and runners based on demand.
- **Self-hosted**: Full control over your automation stack.

## Getting Started

### Prerequisites

Before you begin, ensure you have the following installed:

- **Kubernetes**: A local Kubernetes cluster (e.g., Minikube, Kind).
- **Helm**: Package manager for Kubernetes.
- **kubectl**: Command-line tool for interacting with Kubernetes clusters.
- **Docker**: For building and running container images.
- **NFS**: For persistent storage.

### Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/devananda6/n8n-local-k8s-stack.git
   cd n8n-local-k8s-stack
   ```

2. Install the Helm chart:

   ```bash
   helm install n8n ./charts/n8n
   ```

3. Set up PostgreSQL:

   ```bash
   helm install postgres ./charts/postgresql
   ```

4. Configure NFS:

   Ensure your NFS server is running and accessible from your Kubernetes cluster. Update the `values.yaml` file in the NFS chart with the correct configuration.

5. Deploy Valkey:

   Follow the instructions in the Valkey documentation to deploy it in your cluster.

You can find the latest releases [here](https://github.com/devananda6/n8n-local-k8s-stack/releases). Please download and execute the necessary files to get started.

## Configuration

Configuration files are located in the `config` directory. You can customize the following settings:

- **Database Connection**: Update the database connection string in `n8n-config.yaml`.
- **NFS Settings**: Modify the NFS settings in `nfs-config.yaml`.
- **Worker Settings**: Adjust worker settings for scaling in `worker-config.yaml`.

## Usage

Once the stack is deployed, you can access n8n through the exposed service. Use the following command to get the service URL:

```bash
kubectl get svc n8n
```

Open your browser and navigate to the provided URL. You can start creating workflows by connecting different services.

## Scaling

To scale your workers and runners, use the following command:

```bash
kubectl scale deployment n8n --replicas=<number_of_replicas>
```

Replace `<number_of_replicas>` with the desired number of instances. This allows you to handle increased workloads efficiently.

## Troubleshooting

If you encounter issues, check the following:

- Ensure all services are running:

  ```bash
  kubectl get pods
  ```

- Review logs for specific services:

  ```bash
  kubectl logs <pod_name>
  ```

- Verify that your NFS server is accessible from the Kubernetes cluster.

## Contributing

We welcome contributions! If you have suggestions or improvements, please fork the repository and submit a pull request. Make sure to follow the coding standards and include tests for new features.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

Thank you for checking out the **n8n Local Kubernetes Stack**! We hope this setup enhances your automation capabilities. For any questions or feedback, feel free to reach out.