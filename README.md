# DOC
DEP-MGPU
Лабораторная работа: Миграция Docker Compose в Kubernetes
Вариант 2 (Финансы - Crypto Dashboard) с реализацией Resource Quotas
1. Исходное приложение (Docker Compose)
Структура проекта до миграции
text
crypto-app/
├── docker-compose.yml
├── db/
│   └── init.sql
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
└── loader/
    ├── loader.py
    ├── requirements.txt
    └── Dockerfile
docker-compose.yml (исходный)
yaml
version: '3.8'

services:
  db:
    image: crypto-db:latest
    build: ./db
    container_name: crypto-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - crypto-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    image: crypto-app:latest
    build: ./app
    container_name: crypto-dashboard
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DB_NAME: ${POSTGRES_DB}
      API_KEY: ${CRYPTO_API_KEY}
      LOG_LEVEL: INFO
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - crypto-network

  loader:
    image: crypto-loader:latest
    build: ./loader
    container_name: crypto-loader
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DB_NAME: ${POSTGRES_DB}
      API_KEY: ${CRYPTO_API_KEY}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - crypto-network
    restart: "no"

volumes:
  postgres_data:

networks:
  crypto-network:
    driver: bridge
.env файл (исходный)
env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=securepassword123
POSTGRES_DB=crypto_db
CRYPTO_API_KEY=coinmarketcap_api_key_12345
2. Код приложения
app/app.py (Python Flask приложение)
python
import os
import time
import logging
import psycopg2
from flask import Flask, jsonify, render_template_string
import requests
from datetime import datetime

# Настройка логирования
logging.basicConfig(level=os.getenv('LOG_LEVEL', 'INFO'))
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Конфигурация из переменных окружения
DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_PORT = os.getenv('DB_PORT', '5432')
DB_USER = os.getenv('DB_USER', 'postgres')
DB_PASSWORD = os.getenv('DB_PASSWORD', '')
DB_NAME = os.getenv('DB_NAME', 'crypto_db')
API_KEY = os.getenv('API_KEY', '')
DASHBOARD_TITLE = os.getenv('DASHBOARD_TITLE', 'Crypto Dashboard')

def get_db_connection():
    """Создание подключения к базе данных"""
    try:
        conn = psycopg2.connect(
            host=DB_HOST,
            port=DB_PORT,
            user=DB_USER,
            password=DB_PASSWORD,
            database=DB_NAME
        )
        return conn
    except Exception as e:
        logger.error(f"Ошибка подключения к БД: {e}")
        return None

def init_db():
    """Инициализация таблиц, если их нет"""
    conn = get_db_connection()
    if conn:
        cur = conn.cursor()
        cur.execute('''
            CREATE TABLE IF NOT EXISTS cryptocurrencies (
                id SERIAL PRIMARY KEY,
                symbol VARCHAR(10) NOT NULL,
                name VARCHAR(100) NOT NULL,
                price DECIMAL(20, 8),
                volume_24h DECIMAL(20, 2),
                market_cap DECIMAL(20, 2),
                last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        conn.commit()
        cur.close()
        conn.close()
        logger.info("База данных инициализирована")

# HTML шаблон для дашборда
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f0f2f5; }
        .container { max-width: 1200px; margin: 0 auto; }
        .header { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); 
                  color: white; padding: 20px; border-radius: 10px; margin-bottom: 20px; }
        .crypto-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 20px; }
        .crypto-card { background: white; border-radius: 10px; padding: 20px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .crypto-symbol { font-size: 24px; font-weight: bold; color: #333; }
        .crypto-name { color: #666; margin: 5px 0; }
        .crypto-price { font-size: 28px; color: #28a745; margin: 10px 0; }
        .crypto-volume { color: #17a2b8; }
        .crypto-market-cap { color: #6c757d; }
        .update-time { text-align: center; color: #999; margin-top: 20px; font-size: 12px; }
        .pod-info { background: #e9ecef; padding: 10px; border-radius: 5px; margin-top: 10px; font-size: 12px; }
        .health-endpoints { margin-top: 20px; padding: 10px; background: #d4edda; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>{{ title }}</h1>
            <p>Данные криптовалют в реальном времени</p>
        </div>
        
        <div class="crypto-grid">
            {% for crypto in cryptos %}
            <div class="crypto-card">
                <div class="crypto-symbol">{{ crypto.symbol }}</div>
                <div class="crypto-name">{{ crypto.name }}</div>
                <div class="crypto-price">${{ "%.2f"|format(crypto.price) }}</div>
                <div class="crypto-volume">Объем (24ч): ${{ "%.0f"|format(crypto.volume_24h) }}</div>
                <div class="crypto-market-cap">Капитализация: ${{ "%.0f"|format(crypto.market_cap) }}</div>
            </div>
            {% endfor %}
        </div>
        
        <div class="pod-info">
            <strong>Информация о поде:</strong><br>
            Имя пода: {{ pod_name }}<br>
            IP пода: {{ pod_ip }}<br>
            Нода: {{ node_name }}<br>
            Временная метка: {{ timestamp }}
        </div>
        
        <div class="health-endpoints">
            <strong>Endpoints для проверки здоровья:</strong><br>
            /health - Liveness Probe<br>
            /ready - Readiness Probe
        </div>
        
        <div class="update-time">
            Данные обновлены: {{ timestamp }}
        </div>
    </div>
</body>
</html>
'''

@app.route('/')
def dashboard():
    """Главная страница дашборда"""
    try:
        conn = get_db_connection()
        cryptos = []
        if conn:
            cur = conn.cursor()
            cur.execute('SELECT symbol, name, price, volume_24h, market_cap FROM cryptocurrencies ORDER BY market_cap DESC')
            cryptos = [{'symbol': row[0], 'name': row[1], 'price': float(row[2]), 
                       'volume_24h': float(row[3]), 'market_cap': float(row[4])} 
                      for row in cur.fetchall()]
            cur.close()
            conn.close()
        
        # Получаем информацию о подe из Downward API
        pod_name = os.getenv('POD_NAME', 'unknown')
        pod_ip = os.getenv('POD_IP', 'unknown')
        node_name = os.getenv('NODE_NAME', 'unknown')
        
        return render_template_string(
            HTML_TEMPLATE,
            title=DASHBOARD_TITLE,
            cryptos=cryptos,
            pod_name=pod_name,
            pod_ip=pod_ip,
            node_name=node_name,
            timestamp=datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        )
    except Exception as e:
        logger.error(f"Ошибка в dashboard: {e}")
        return f"Ошибка: {e}", 500

@app.route('/health')
def health():
    """Liveness Probe - проверка, жив ли контейнер"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat()
    }), 200

@app.route('/ready')
def ready():
    """Readiness Probe - готов ли принимать трафик"""
    # Проверяем подключение к БД
    conn = get_db_connection()
    if conn:
        conn.close()
        return jsonify({
            'status': 'ready',
            'database': 'connected',
            'timestamp': datetime.now().isoformat()
        }), 200
    else:
        return jsonify({
            'status': 'not ready',
            'database': 'disconnected',
            'timestamp': datetime.now().isoformat()
        }), 503

@app.route('/api/cryptos')
def api_cryptos():
    """API для получения данных в JSON"""
    conn = get_db_connection()
    if not conn:
        return jsonify({'error': 'Database connection failed'}), 503
    
    cur = conn.cursor()
    cur.execute('SELECT * FROM cryptocurrencies')
    data = [{'id': row[0], 'symbol': row[1], 'name': row[2], 'price': float(row[3]),
             'volume_24h': float(row[4]), 'market_cap': float(row[5]), 'last_updated': row[6].isoformat()}
            for row in cur.fetchall()]
    cur.close()
    conn.close()
    
    return jsonify(data)

if __name__ == '__main__':
    # Инициализация БД при старте
    init_db()
    # Запуск приложения
    app.run(host='0.0.0.0', port=5000, debug=False)
app/requirements.txt
text
Flask==2.3.3
psycopg2-binary==2.9.7
requests==2.31.0
app/Dockerfile
dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
loader/loader.py (загрузчик данных)
python
import os
import time
import logging
import psycopg2
import requests
from datetime import datetime

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Конфигурация из переменных окружения
DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_PORT = os.getenv('DB_PORT', '5432')
DB_USER = os.getenv('DB_USER', 'postgres')
DB_PASSWORD = os.getenv('DB_PASSWORD', '')
DB_NAME = os.getenv('DB_NAME', 'crypto_db')
API_KEY = os.getenv('API_KEY', '')

# Тестовые данные криптовалют (для демонстрации, в реальности используется API)
MOCK_CRYPTOS = [
    {'symbol': 'BTC', 'name': 'Bitcoin', 'price': 50000, 'volume_24h': 30000000000, 'market_cap': 950000000000},
    {'symbol': 'ETH', 'name': 'Ethereum', 'price': 3000, 'volume_24h': 15000000000, 'market_cap': 360000000000},
    {'symbol': 'BNB', 'name': 'Binance Coin', 'price': 400, 'volume_24h': 2000000000, 'market_cap': 62000000000},
    {'symbol': 'SOL', 'name': 'Solana', 'price': 100, 'volume_24h': 1500000000, 'market_cap': 42000000000},
    {'symbol': 'ADA', 'name': 'Cardano', 'price': 0.5, 'volume_24h': 800000000, 'market_cap': 17500000000},
    {'symbol': 'DOT', 'name': 'Polkadot', 'price': 7, 'volume_24h': 300000000, 'market_cap': 8500000000},
]

def get_db_connection():
    """Создание подключения к базе данных с повторными попытками"""
    max_retries = 10
    retry_delay = 3
    
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(
                host=DB_HOST,
                port=DB_PORT,
                user=DB_USER,
                password=DB_PASSWORD,
                database=DB_NAME,
                connect_timeout=5
            )
            logger.info(f"Подключение к БД успешно (попытка {attempt + 1})")
            return conn
        except Exception as e:
            logger.warning(f"Попытка {attempt + 1}/{max_retries} подключения к БД не удалась: {e}")
            if attempt < max_retries - 1:
                time.sleep(retry_delay)
            else:
                logger.error("Не удалось подключиться к БД после всех попыток")
                raise

def fetch_crypto_data():
    """Получение данных о криптовалютах (мок-данные)"""
    # В реальном проекте здесь был бы запрос к API
    # response = requests.get(f'https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest', 
    #                        headers={'X-CMC_PRO_API_KEY': API_KEY})
    # return response.json()
    
    logger.info("Используются тестовые данные")
    return MOCK_CRYPTOS

def load_data_to_db():
    """Загрузка данных в базу"""
    try:
        logger.info("Начало загрузки данных в БД")
        
        # Подключение к БД
        conn = get_db_connection()
        cur = conn.cursor()
        
        # Очистка таблицы
        cur.execute('TRUNCATE cryptocurrencies RESTART IDENTITY')
        logger.info("Таблица очищена")
        
        # Получение данных
        cryptos = fetch_crypto_data()
        
        # Вставка данных
        for crypto in cryptos:
            cur.execute('''
                INSERT INTO cryptocurrencies (symbol, name, price, volume_24h, market_cap)
                VALUES (%s, %s, %s, %s, %s)
            ''', (
                crypto['symbol'],
                crypto['name'],
                crypto['price'],
                crypto['volume_24h'],
                crypto['market_cap']
            ))
        
        conn.commit()
        logger.info(f"Успешно загружено {len(cryptos)} записей")
        
        cur.close()
        conn.close()
        
        return True
        
    except Exception as e:
        logger.error(f"Ошибка при загрузке данных: {e}")
        return False

def main():
    """Основная функция"""
    logger.info("Запуск загрузчика данных")
    logger.info(f"Параметры подключения: {DB_HOST}:{DB_PORT}, БД: {DB_NAME}")
    
    # Ждем немного, чтобы БД точно была готова
    time.sleep(5)
    
    success = load_data_to_db()
    
    if success:
        logger.info("Загрузка данных завершена успешно")
        exit(0)
    else:
        logger.error("Загрузка данных завершилась с ошибкой")
        exit(1)

if __name__ == "__main__":
    main()
loader/requirements.txt
text
psycopg2-binary==2.9.7
requests==2.31.0
loader/Dockerfile
dockerfile
FROM python:3.9-slim

WORKDIR /loader

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY loader.py .

CMD ["python", "loader.py"]
db/init.sql
sql
-- Создание таблицы криптовалют
CREATE TABLE IF NOT EXISTS cryptocurrencies (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(10) NOT NULL,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(20, 8),
    volume_24h DECIMAL(20, 2),
    market_cap DECIMAL(20, 2),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Индекс для быстрого поиска
CREATE INDEX IF NOT EXISTS idx_symbol ON cryptocurrencies(symbol);
db/Dockerfile
dockerfile
FROM postgres:15

# Копирование скрипта инициализации
COPY init.sql /docker-entrypoint-initdb.d/

# Настройки PostgreSQL
ENV POSTGRES_USER=postgres
ENV POSTGRES_PASSWORD=postgres
ENV POSTGRES_DB=crypto_db

EXPOSE 5432
3. Манифесты Kubernetes
Создаем директорию для манифестов
bash
mkdir -p crypto-k8s/k8s
00-namespace.yaml
yaml
apiVersion: v1
kind: Namespace
metadata:
  name: crypto-dashboard
  labels:
    name: crypto-dashboard
    environment: development
    project: crypto-finance
01-secret.yaml
yaml
apiVersion: v1
kind: Secret
metadata:
  name: crypto-secrets
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: secrets
type: Opaque
data:
  # echo -n "postgres" | base64
  POSTGRES_USER: cG9zdGdyZXM=
  # echo -n "securepassword123" | base64
  POSTGRES_PASSWORD: c2VjdXJlcGFzc3dvcmQxMjM=
  # echo -n "crypto_db" | base64
  POSTGRES_DB: Y3J5cHRvX2Ri
  # echo -n "coinmarketcap_api_key_12345" | base64
  API_KEY: Y29pbm1hcmtldGNhcF9hcGlfa2V5XzEyMzQ1
02-configmap.yaml
yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: crypto-config
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: config
data:
  # Настройки приложения
  DB_HOST: "crypto-db-service"
  DB_PORT: "5432"
  LOG_LEVEL: "INFO"
  DASHBOARD_TITLE: "Crypto Dashboard K8s"
  REFRESH_INTERVAL: "30"
  
  # Настройки базы данных
  POSTGRES_MAX_CONNECTIONS: "100"
  POSTGRES_SHARED_BUFFERS: "128MB"
  
  # Конфигурационные файлы (можно монтировать как файлы)
  app-config.json: |
    {
      "theme": "dark",
      "refreshRate": 30,
      "currencies": ["USD", "EUR", "BTC"],
      "features": {
        "charts": true,
        "alerts": true,
        "news": false
      }
    }
  
  # Логирование
  logging.conf: |
    [loggers]
    keys=root
    
    [handlers]
    keys=consoleHandler
    
    [formatters]
    keys=simpleFormatter
    
    [logger_root]
    level=INFO
    handlers=consoleHandler
    
    [handler_consoleHandler]
    class=StreamHandler
    level=INFO
    formatter=simpleFormatter
    args=(sys.stdout,)
    
    [formatter_simpleFormatter]
    format=%(asctime)s - %(name)s - %(levelname)s - %(message)s
    datefmt=
03-pvc.yaml
yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: crypto-db-pvc
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: database
    type: persistent-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard
04-resource-quota.yaml (специфика варианта 2)
yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: crypto-resource-quota
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: quota
spec:
  hard:
    # Лимиты на запрашиваемые ресурсы
    requests.cpu: "2"
    requests.memory: "2Gi"
    
    # Лимиты на максимальное потребление
    limits.cpu: "4"
    limits.memory: "4Gi"
    
    # Лимиты на количество объектов
    persistentvolumeclaims: "2"
    pods: "10"
    services: "5"
    secrets: "10"
    configmaps: "10"
    
    # Лимиты на хранилище
    requests.storage: "5Gi"
05-db-deployment.yaml
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crypto-db
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: database
    environment: dev
    tier: backend
    version: v1
  annotations:
    description: "PostgreSQL база данных для Crypto Dashboard"
    maintainer: "DevOps Team"
spec:
  replicas: 1
  strategy:
    type: Recreate  # Для БД используем Recreate, т.к. не можем иметь два экземпляра с одним PVC
  selector:
    matchLabels:
      app: crypto
      component: database
  template:
    metadata:
      labels:
        app: crypto
        component: database
        environment: dev
        tier: backend
        version: v1
    spec:
      containers:
      - name: postgres
        image: crypto-db:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5432
          name: postgres
        env:
        # Secrets
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_DB
        # ConfigMap
        - name: POSTGRES_MAX_CONNECTIONS
          valueFrom:
            configMapKeyRef:
              name: crypto-config
              key: POSTGRES_MAX_CONNECTIONS
        - name: POSTGRES_SHARED_BUFFERS
          valueFrom:
            configMapKeyRef:
              name: crypto-config
              key: POSTGRES_SHARED_BUFFERS
        # Downward API - информация о поде
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
          subPath: postgres-data  # Чтобы не перезаписывать всю директорию
        - name: postgres-init
          mountPath: /docker-entrypoint-initdb.d
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
        startupProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 1
          periodSeconds: 2
          failureThreshold: 30  # 60 секунд на запуск
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: crypto-db-pvc
      - name: postgres-init
        configMap:
          name: crypto-config
          items:
          - key: init.sql
            path: init.sql
            mode: 0644
06-db-service.yaml
yaml
apiVersion: v1
kind: Service
metadata:
  name: crypto-db-service
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: database
    service: postgres
spec:
  selector:
    app: crypto
    component: database
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
    name: postgres
  type: ClusterIP  # Внутренний сервис, доступен только внутри кластера
07-app-deployment.yaml
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crypto-app
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: dashboard
    environment: dev
    tier: frontend
    version: v1
  annotations:
    description: "Flask приложение для отображения криптовалют"
    prometheus.io/scrape: "true"
    prometheus.io/port: "5000"
spec:
  replicas: 2  # Две реплики для балансировки нагрузки
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Гарантируем, что всегда будет доступен хотя бы один под
  selector:
    matchLabels:
      app: crypto
      component: dashboard
  template:
    metadata:
      labels:
        app: crypto
        component: dashboard
        environment: dev
        tier: frontend
        version: v1
    spec:
      # InitContainer для проверки доступности БД
      initContainers:
      - name: wait-for-db
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Ожидание доступности базы данных..."
          until nc -z -v -w5 crypto-db-service 5432; do
            echo "База данных не доступна - ожидание 2 секунды..."
            sleep 2
          done
          echo "База данных доступна!"
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: crypto-config
              key: DB_HOST
      
      containers:
      - name: crypto-app
        image: crypto-app:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
          name: http
        # Загрузка всех переменных из ConfigMap
        envFrom:
        - configMapRef:
            name: crypto-config
        # Индивидуальные переменные из Secrets
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_PASSWORD
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_DB
        - name: CRYPTO_API_KEY
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: API_KEY
        # Downward API - информация о поде
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        # Liveness Probe - проверка живости приложения
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        # Readiness Probe - проверка готовности принимать трафик
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
            scheme: HTTP
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: logging-config
          mountPath: /app/logging.conf
          subPath: logging.conf
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: crypto-config
          items:
          - key: app-config.json
            path: app-config.json
      - name: logging-config
        configMap:
          name: crypto-config
          items:
          - key: logging.conf
            path: logging.conf
08-app-service.yaml
yaml
apiVersion: v1
kind: Service
metadata:
  name: crypto-app-service
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: dashboard
    service: web
spec:
  selector:
    app: crypto
    component: dashboard
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30007  # Фиксированный NodePort (для варианта 3)
    protocol: TCP
    name: http
  type: NodePort  # Доступ снаружи кластера
09-loader-job.yaml
yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: crypto-loader
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: loader
    job-type: one-time
  annotations:
    description: "Job для загрузки начальных данных в БД"
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 4  # Количество повторных попыток при ошибке
  activeDeadlineSeconds: 300  # Максимальное время выполнения (5 минут)
  ttlSecondsAfterFinished: 3600  # Удалить через час после завершения
  template:
    metadata:
      labels:
        app: crypto
        component: loader
    spec:
      restartPolicy: Never
      containers:
      - name: loader
        image: crypto-loader:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: crypto-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: crypto-config
              key: DB_PORT
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_PASSWORD
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: POSTGRES_DB
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: crypto-secrets
              key: API_KEY
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
10-hpa.yaml (опционально - Horizontal Pod Autoscaler)
yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: crypto-app-hpa
  namespace: crypto-dashboard
  labels:
    app: crypto
    component: dashboard
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: crypto-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
4. Скрипты для развертывания
deploy.sh
bash
#!/bin/bash

set -e

echo "======================================"
echo "Развертывание Crypto Dashboard в Kubernetes"
echo "======================================"

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Проверка наличия minikube
if ! command -v minikube &> /dev/null; then
    echo -e "${RED}minikube не установлен. Установите minikube и повторите попытку.${NC}"
    exit 1
fi

# Проверка наличия kubectl
if ! command -v kubectl &> /dev/null; then
    echo -e "${RED}kubectl не установлен. Установите kubectl и повторите попытку.${NC}"
    exit 1
fi

echo -e "${YELLOW}1. Запуск Minikube
