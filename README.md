# EC2 Logs + AWS CloudWatchLogs


Configuração para envio de logs de uma instância EC2 para o Amazon CloudWatch Logs. 


É possível centralizar todos os logs de serviços e sistema em um único local, facilitando a monitorização e análise.

---

## Pré-requistos

>[!IMPORTANT]
>Habilite o Detailed Monitoring no Amazon CloudWatch ao lançar sua instância EC2, acessando o menu Additional Settings durante o processo de configuração.


![00](https://github.com/user-attachments/assets/ebe7680e-9de2-44a4-af5b-ffc2e8197eb9)


## 1. Configurar a Role com Permissões para CloudWatch Logs

1. No Console AWS:
   - Vá para **IAM** > **Roles**.
   - Clique em **Create role**.
   - Escolha **AWS Service** > **EC2** e clique em **Next**.
   - Anexe a política gerenciada **CloudWatchAgentServerPolicy**.
   - Nomeie a Role como `EC2CloudWatchLogsRole` e clique em **Create role**.


![1](https://github.com/user-attachments/assets/175358bf-6de3-4fd0-a44d-06aaaeda1df5)


![2](https://github.com/user-attachments/assets/3e0744de-3454-4807-8c4d-df6c876c1507)


2. Associe a Role à sua instância EC2:
   - No Console AWS, vá para **EC2** > **Instances**.
   - Selecione sua instância, clique em **Actions** > **Security** > **Modify IAM Role**.
   - Escolha a Role criada (`EC2CloudWatchLogsRole`) e salve.


![3](https://github.com/user-attachments/assets/6eac41a0-e1ea-4c9c-b2b6-f3d7320c4c43)


![4](https://github.com/user-attachments/assets/428a7f59-6ed9-43c1-8a82-32d448c6c56e)


---

## 2. Instalar o CloudWatch Agent

1. Conecte-se à sua instância EC2 via SSH.

2. Atualize os pacotes da instância:  
   ```bash
   sudo yum update -y
   ```

3. Instale o CloudWatch Agent
   ```bash
   sudo yum install amazon-cloudwatch-agent -y
   ```


![5](https://github.com/user-attachments/assets/c4408883-5c99-477f-92c4-53645e3e93ad)


---

## 3. Configurar o CloudWatch Agent

1. Crie o arquivo de configuração:
>[!NOTE]
>O agente usa um arquivo .json para definir quais logs coletar e para onde enviá-los.

- crie o arquivo em `/opt/aws/amazon-cloudwatch-agent/bin/config.json`:
  ```bash
  sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
  ```

2. Adicione a configuração básica abaixo:
```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "EC2-Logs",
            "log_stream_name": "{instance_id}/messages",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/cloud-init.log",
            "log_group_name": "EC2-Logs",
            "log_stream_name": "{instance_id}/cloud-init",
            "timezone": "UTC"
          }
        ]
      }
    }
  }
}
```

- `file_path`: Caminho para o arquivo de log que você deseja monitorar.
- `log_group_name`: Nome do grupo de logs no CloudWatch (será criado automaticamente, se não existir).
- `log_stream_name`: Nome do stream no grupo de logs.


![6](https://github.com/user-attachments/assets/1f957aa8-beec-4c3b-b585-7416c32bf03b)


---

## 4. Iniciar o CloudWatch Agent

1. Aplicar a configuração do agente:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
    -s
```


![7](https://github.com/user-attachments/assets/49502f1b-db65-4314-bfc4-c35bc94da6d6)


2. Verifique o status do agente:
```bash
sudo systemctl status amazon-cloudwatch-agent
```


![8](https://github.com/user-attachments/assets/5c27d88a-3392-4a68-bea1-8400c3765e3d)


---

## 5. Validar os Logs no CloudWatch

1. No Console AWS:
   - Vá para **CloudWatch** > **Logs** > **Log groups**.
   - Localize o grupo de logs configurado (por exemplo, `EC2-Logs`).

2. Clique no grupo de logs para visualizar os streams:
   - Você verá os logs provenientes da sua instância EC2, organizados em streams como `{instance_id}/messages`, `{instance_id}/cloud-init`, e outros dependendo da configuração.

3. Se os logs não aparecerem imediatamente:
   - Verifique o status do CloudWatch Agent na sua instância EC2:
     ```bash
     sudo systemctl status amazon-cloudwatch-agent
     ```
   - Certifique-se de que o agente está rodando sem erros.


![404547952-a4de45b8-faec-49b9-9ef5-bdc8c36fa447](https://github.com/user-attachments/assets/a968537a-d8e0-4f8f-985c-f51bffba1e4e)


![404548031-25f81ade-5c51-4236-a7c0-b62e13fb58d4](https://github.com/user-attachments/assets/11570811-40aa-4b38-91b5-5c9229919b18)


![11](https://github.com/user-attachments/assets/86eb4bdc-6ec6-489b-beb5-b9a0ee83131f)


---

## 6. Configuração (Opcional)

Se você deseja monitorar mais arquivos de log ou configurar o agente de forma mais específica, basta editar o arquivo de configuração do CloudWatch Agent. Abaixo estão algumas sugestões de logs adicionais que você pode configurar:

- **Logs de acesso do Apache:**
```json
{
  "file_path": "/var/log/httpd/access_log",
  "log_group_name": "WebServer-Logs",
  "log_stream_name": "{instance_id}/apache-access",
  "timezone": "UTC"
}
```

- **Logs de erro do Apache:**
```json
{
  "file_path": "/var/log/httpd/error_log",
  "log_group_name": "WebServer-Logs",
  "log_stream_name": "{instance_id}/apache-error",
  "timezone": "UTC"
}
```

- **Logs de acesso do Nginx**:
```json
{
  "file_path": "/var/log/nginx/access.log",
  "log_group_name": "WebServer-Logs",
  "log_stream_name": "{instance_id}/nginx-access",
  "timezone": "UTC"
}
```

- **Logs de erro do Nginx**:
```json
{
  "file_path": "/var/log/nginx/error.log",
  "log_group_name": "WebServer-Logs",
  "log_stream_name": "{instance_id}/nginx-error",
  "timezone": "UTC"
}
```

- **Logs de erro do MySQL**:
```json
{
  "file_path": "/var/log/mysql/error.log",
  "log_group_name": "Database-Logs",
  "log_stream_name": "{instance_id}/mysql-error",
  "timezone": "UTC"
}
```

- **Logs de erro do PostgreSQL**:
```json
{
  "file_path": "/var/log/postgresql/postgresql.log",
  "log_group_name": "Database-Logs",
  "log_stream_name": "{instance_id}/postgresql-error",
  "timezone": "UTC"
}
```

- **Logs de segurança**: 
```json
{
  "file_path": "/var/log/secure",
  "log_group_name": "Security-Logs",
  "log_stream_name": "{instance_id}/secure-log",
  "timezone": "UTC"
}
```

- Após adicionar ou modificar as configurações, aplique novamente a configuração do agente:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
    -s
```

---

## Exemplo Completo de Configuração

Se você quiser configurar múltiplos logs, o arquivo config.json ficaria assim:
```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "EC2-Logs",
            "log_stream_name": "{instance_id}/messages",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "EC2-Logs",
            "log_stream_name": "{instance_id}/syslog",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "WebServer-Logs",
            "log_stream_name": "{instance_id}/nginx-access",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/secure",
            "log_group_name": "Security-Logs",
            "log_stream_name": "{instance_id}/secure-log",
            "timezone": "UTC"
          }
        ]
      }
    }
  }
}
```

>[!TIP]
>  Caminhos de log personalizados: Se você estiver usando um caminho diferente ou se sua aplicação gravar logs em arquivos específicos, basta adicionar esses caminhos no arquivo de configuração.
>
>  Vários logs: O collect_list pode conter múltiplos arquivos de log, como mostrado no exemplo. Basta adicionar quantos arquivos forem necessários.

Com essa configuração, o CloudWatch Agent irá coletar e enviar os logs especificados para o CloudWatch Logs, onde você poderá monitorá-los e configurar alarmes ou métricas conforme necessário.

