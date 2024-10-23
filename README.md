# Guia Completo para Integração do MinIO com Airflow para Salvar e Ler Arquivos CSV

Este guia fornece um passo a passo para configurar o MinIO juntamente com o Apache Airflow, com o objetivo de salvar um arquivo CSV gerado por uma DAG e posteriormente utilizá-lo em um processo de web scraping.

## 1. Configurando o MinIO no Docker Compose

Edite o arquivo `docker-compose.yml` para incluir o serviço MinIO. O MinIO é um armazenamento de objetos compatível com S3 e será utilizado para salvar e ler o arquivo CSV.

### Passos para Adicionar o Serviço MinIO:
1. Abra o arquivo `docker-compose.yml` em um editor de texto.
2. Adicione a configuração do MinIO conforme o exemplo abaixo:

```yaml
version: '3'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:latest
    ports:
      - "6379:6379"

  webserver:
    image: apache/airflow:2.5.0
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      GOOGLE_APPLICATION_CREDENTIALS: /opt/airflow/credentials/bigquery_keyfile.json
    ports:
      - "8080:8080"
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./credentials:/opt/airflow/credentials
    command: bash -c "pip install apache-airflow-providers-google requests beautifulsoup4 pandas && airflow db upgrade && exec airflow webserver"
    depends_on:
      - postgres
      - redis
      - minio

  scheduler:
    image: apache/airflow:2.5.0
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      GOOGLE_APPLICATION_CREDENTIALS: /opt/airflow/credentials/bigquery_keyfile.json
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./credentials:/opt/airflow/credentials
    command: bash -c "pip install apache-airflow-providers-google requests beautifulsoup4 pandas && exec airflow scheduler"
    depends_on:
      - postgres
      - redis
      - minio

  worker:
    image: apache/airflow:2.5.0
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      GOOGLE_APPLICATION_CREDENTIALS: /opt/airflow/credentials/bigquery_keyfile.json
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./credentials:/opt/airflow/credentials
    command: bash -c "pip install apache-airflow-providers-google requests beautifulsoup4 pandas && exec airflow celery worker"
    depends_on:
      - postgres
      - redis
      - minio

  minio:
    image: minio/minio
    environment:
      MINIO_ACCESS_KEY: "admin"
      MINIO_SECRET_KEY: "admin"
    ports:
      - "9000:9000"
    command: server /data
    volumes:
      - ./minio-data:/data

networks:
  default:
    driver: bridge
```

## 2. Criar o Bucket no MinIO

Após configurar o MinIO no Docker Compose, siga os passos abaixo para criar um bucket no MinIO:

1. **Acesse o MinIO**: Abra um navegador e vá para `http://localhost:9000`.
2. **Login**: Faça login usando as credenciais (`admin` para `ACCESS_KEY` e `admin` para `SECRET_KEY`).
3. **Criar um Novo Bucket**:
   - Clique em **+ Create Bucket** no painel do MinIO.
   - Escolha um nome para o bucket, como `meu-bucket`, e clique em **Create Bucket**.

## 3. Ajustar o Código Python para Salvar no MinIO

Para salvar o arquivo CSV gerado pela DAG diretamente no MinIO, você precisará usar a biblioteca `boto3`.

### Passos:

1. **Instalar o `boto3` no Ambiente do Airflow**

   Acesse o terminal do container do Airflow e execute o comando abaixo para instalar o `boto3`:

   ```bash
   pip install boto3
   ```

2. **Atualizar a Função para Salvar o CSV no MinIO**

   Atualize a função `save_links_to_csv` para salvar o CSV no MinIO:

   ```python
   import boto3
   from botocore.client import Config
   import csv

   # Configurar cliente MinIO
   minio_client = boto3.client(
       's3',
       endpoint_url='http://localhost:9000',
       aws_access_key_id='admin',
       aws_secret_access_key='admin',
       config=Config(signature_version='s3v4'),
       region_name='us-east-1'
   )

   # Salvar os links em um arquivo CSV e fazer o upload para o MinIO
   def save_links_to_csv(urls, bucket_name='meu-bucket', object_name='event_links.csv'):
       # Salvar localmente
       filename = '/tmp/event_links.csv'
       try:
           with open(filename, mode='w', newline='') as file:
               writer = csv.writer(file)
               writer.writerow(["Links dos Eventos"])
               for link in urls:
                   writer.writerow([link])
           print(f"Arquivo CSV '{filename}' criado com sucesso!")
       except IOError as e:
           print(f"Erro ao salvar o arquivo CSV: {e}")
           return

       # Fazer upload para o MinIO
       try:
           minio_client.upload_file(filename, bucket_name, object_name)
           print(f"Arquivo CSV '{object_name}' enviado ao MinIO com sucesso!")
       except Exception as e:
           print(f"Erro ao fazer upload para o MinIO: {e}")
   ```

## 4. Ler o Arquivo CSV do MinIO para o Scraping

Agora, ajuste o código para ler o arquivo CSV diretamente do MinIO antes de realizar o scraping.

### Passos:

1. **Configurar o Cliente MinIO** e **Fazer o Download do Arquivo CSV**:

   Atualize o código Python para incluir a funcionalidade de baixar o arquivo CSV do MinIO:
   ```python
   import boto3
   import csv
   from botocore.client import Config

   # Configurar cliente MinIO
   minio_client = boto3.client(
       's3',
       endpoint_url='http://localhost:9000',
       aws_access_key_id='admin',
       aws_secret_access_key='admin',
       config=Config(signature_version='s3v4'),
       region_name='us-east-1'
   )

   # Fazer o download do arquivo CSV do MinIO
   def download_csv_from_minio(bucket_name='meu-bucket', object_name='event_links.csv', download_path='/tmp/event_links.csv'):
       try:
           minio_client.download_file(bucket_name, object_name, download_path)
           print(f"Arquivo CSV '{object_name}' baixado com sucesso do MinIO!")
           return download_path
       except Exception as e:
           print(f"Erro ao baixar o arquivo CSV do MinIO: {e}")
           return None

   # Ler as URLs do arquivo CSV
   downloaded_file_path = download_csv_from_minio()
   urls = []
   if downloaded_file_path:
       try:
           with open(downloaded_file_path, mode='r') as file:
               reader = csv.reader(file)
               next(reader)  # Pular o cabeçalho
               for row in reader:
                   urls.append(row[0])
       except IOError as e:
           print(f"Erro ao ler o arquivo CSV: {e}")
   ```

## 5. Resumo do Processo

1. **Adicionar o MinIO ao Docker Compose**: Configure o serviço MinIO para funcionar junto com o Airflow.
   - Adicione o serviço `minio` ao arquivo Docker Compose, incluindo portas e volumes para persistência de dados.
2. **Criar o Bucket no MinIO**: Acesse a interface web do MinIO e crie um bucket para armazenar os arquivos.
   - Use as credenciais definidas para fazer login (`admin` para `ACCESS_KEY` e `admin` para `SECRET_KEY`) e crie um bucket chamado `meu-bucket`.
3. **Salvar o CSV no MinIO**: Atualize a DAG para salvar o arquivo CSV no MinIO usando o `boto3`.
   - Configure o cliente MinIO e faça o upload do arquivo CSV.
