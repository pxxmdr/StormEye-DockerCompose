# 🚀 Checkpoint 1 - Docker Compose (StormEye)

**Integrantes:**  
- Pedro Henrique Martins dos Reis (RM555306)  
- Adonay Rodrigues da Rocha (RM558782)  
- Thamires Ribeiro (RM558128)  

---

## Repositório do (StormEye) App:
https://github.com/ThamiresRC/StormEye

### 1. Atualizar VM e instalar pacotes básicos
```bash
sudo dnf update -y
sudo dnf install -y git wget nano
```
➡️ Atualiza os pacotes e instala utilitários básicos (git, wget e editor nano).

---

### 2. Instalar Docker e Docker Compose
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
sudo dnf install -y docker-compose-plugin
```
➡️ Instala o Docker Engine e o plugin do Docker Compose. Configura o serviço para iniciar automaticamente e dá permissão ao usuário atual.

---

### 3. Clonar o projeto StormEye
```bash
cd ~
git clone https://github.com/ThamiresRC/StormEye.git app
cd app
```
➡️ Faz o clone do repositório StormEye e acessa a pasta do projeto.

---

### 4. Criar arquivo `.env`
```bash
cat > .env <<'EOF'
MYSQL_DATABASE=stormeye
MYSQL_USER=storm
MYSQL_PASSWORD=stormpwd
MYSQL_ROOT_PASSWORD=rootpwd
EOF
```
➡️ Define variáveis de ambiente usadas pelo banco e pelo app no Docker Compose.

---

### 5. Criar/editar `Dockerfile`
```dockerfile
# Etapa 1: Build
FROM maven:3.9.6-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn -q -DskipTests package

# Etapa 2: Runtime
FROM eclipse-temurin:21-jre
RUN useradd -m app && mkdir -p /home/app/app && chown -R app:app /home/app
WORKDIR /home/app/app
COPY --from=build /app/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=5 \
  CMD sh -c 'exec 3<>/dev/tcp/127.0.0.1/8080 || exit 1'
ENTRYPOINT ["java", "-jar", "app.jar"]
```
➡️ Cria uma imagem multi-stage: primeiro compila com Maven, depois roda o jar com usuário não-root e healthcheck configurado.

---

### 6. Criar `docker-compose.yml`
```yaml
services:
  db:
    image: mysql:8.0
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - appnet
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -p${MYSQL_ROOT_PASSWORD} --silent"]
      interval: 10s
      timeout: 5s
      retries: 15
    restart: unless-stopped

  app:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/${MYSQL_DATABASE}?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
      SPRING_DATASOURCE_USERNAME: ${MYSQL_USER}
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD}
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      SERVER_PORT: 8080
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8080:8080"
    networks:
      - appnet
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "exec 3<>/dev/tcp/localhost/8080"]

  phpmyadmin:
    image: phpmyadmin:latest
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      PMA_USER: ${MYSQL_USER}
      PMA_PASSWORD: ${MYSQL_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - appnet

networks:
  appnet:

volumes:
  db_data:
```
➡️ Define os serviços: banco (MySQL), aplicação (Spring Boot) e interface web (phpMyAdmin), com rede e volume persistente.

---

### 7. Buildar e subir os containers
```bash
docker compose up -d --build
docker compose ps
```
➡️ Faz o build das imagens e sobe os containers em background. Depois lista o status.

---

### 8. Testes
- **API** → http://20.206.106.43:8080/alertas, http://20.206.106.43:8080/catastrofes  
- **phpMyAdmin** → http://20.206.106.43:8081 (login com usuário/senha definidos no `.env`: `storm` / `stormpwd`)

➡️ Permite validar endpoints da aplicação e executar CRUD no banco via phpMyAdmin.

---

# 📊 Testes CRUD no MySQL

Nesta seção estão os comandos SQL utilizados para validar o funcionamento do CRUD no banco de dados `stormeye`, especificamente na tabela **catastrofe**.

---

## ➕ Inserir (CREATE)

```sql
INSERT INTO catastrofe (descricao, localizacao, nivel_gravidade, nome_catastrofe)
VALUES ('Tempestade forte', 'São Paulo', 3, 'Tempestade de verão');
```

---

## 📖 Consultar (READ)

```sql
SELECT * FROM catastrofe;
```

---

## ✏️ Atualizar (UPDATE)

```sql
UPDATE catastrofe
SET nivel_gravidade = 4, descricao = 'Tempestade muito forte'
WHERE id_catastrofe = (Inserir número do ID);
```

---

## ❌ Deletar (DELETE)

```sql
DELETE FROM catastrofe 
WHERE id_catastrofe = (Inserir número do ID);
```

---

📌 **Observação:**  
- O campo `id_catastrofe` é a chave primária da tabela, por isso deve ser usado para identificar o registro correto no momento de atualizar ou deletar.  
- Estes testes podem ser realizados diretamente no **phpMyAdmin** (http://SEU_IP:8081) ou via terminal MySQL dentro do container.
