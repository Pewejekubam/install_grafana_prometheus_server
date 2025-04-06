# Install Grafana and Prometheus Server with TLS

This repository provides a comprehensive solution for setting up a monitoring stack with **Grafana**, **Prometheus**, and **Node Exporter** on Ubuntu servers. It also includes configuration for securing the setup with **Let's Encrypt TLS certificates** using **Nginx** as a reverse proxy.

## Project Overview

The project consists of two main components:

1. **HowTo Documentation**: A step-by-step guide for manually setting up Grafana, Prometheus, and TLS on Ubuntu servers.
2. **Ansible Playbook**: An automated solution for deploying the same setup using Ansible.

Both components are designed to work in parallel, ensuring that users can either follow the manual instructions or use the playbook for automation.

## Features

- **Grafana Installation**: Installs and configures Grafana with a custom configuration file.
- **Prometheus Installation**: Installs Prometheus, sets up its configuration, and ensures it runs as a systemd service.
- **Node Exporter**: Installs Node Exporter for system metrics collection.
- **TLS with Let's Encrypt**: Configures Nginx as a reverse proxy and obtains TLS certificates using Certbot.
- **Ansible Automation**: Automates the entire setup process with idempotent tasks.

## File Structure

- `templates/`: Contains Jinja2 templates for configuration files.
  - `prometheus.yml.j2`: Prometheus configuration template.
  - `grafana.ini.j2`: Grafana configuration template.
  - `certbot_nginx.conf.j2`: Nginx configuration for Certbot.
  - `production_nginx.conf.j2`: Nginx configuration for Grafana.
  - `prometheus.service.j2`: Systemd service file for Prometheus.
  - `node_exporter.service.j2`: Systemd service file for Node Exporter.
- `nodemon_standup.yml`: The main Ansible playbook for automating the setup.
- `LICENSE`: MIT license for the project.

## Prerequisites

- Ubuntu server(s) with sudo privileges.
- A domain name pointing to the server's IP address.
- Ansible installed on the control machine.

## Usage

### Manual Setup

Follow the HowTo documentation to manually install and configure Grafana, Prometheus, and TLS. This approach is ideal for learning and understanding the setup process.

### Automated Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/pewejekubam/install_grafana_prometheus_server.git
   cd install_grafana_prometheus_server
   ```

2. Update the `nodemon_standup.yml` file with your domain name and other variables.

3. Run the playbook:
   ```bash
   ansible-playbook -i inventory nodemon_standup.yml
   ```

## Notes

- The playbook uses Let's Encrypt's staging environment by default to avoid rate limits. Switch to production certificates after testing.
- Ensure that the domain name is correctly configured and resolves to the server's IP address.

## License

This project is licensed under the [MIT License](LICENSE).

## Contributing

Contributions are welcome! Please open an issue or submit a pull request for any improvements or bug fixes.

## Acknowledgments

- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)
- [Let's Encrypt](https://letsencrypt.org/)
