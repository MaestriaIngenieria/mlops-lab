# 🐳 Guía Completa: Dockerizar Modelo ML con FastAPI

Esta guía te muestra cómo dockerizar un modelo de machine learning y exponerlo a través de una API REST usando FastAPI.

## 📋 Tabla de Contenidos

1. [Arquitectura del Sistema](#arquitectura)
2. [Preparación del Modelo](#preparacion)
3. [Desarrollo de la API FastAPI](#fastapi)
4. [Dockerización](#dockerizacion)
5. [Testing y Validación](#testing)

---

## 🏗️ Arquitectura del Sistema {#arquitectura}

```
┌─────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA MLOPS                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  📊 Datos → 🔧 Preprocessing → 🤖 Modelo → 🐳 Docker       │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐    │
│  │   Dataset   │  │ Preprocessor │  │   Trained Model │    │
│  │ (CSV/JSON)  │  │   (.pkl)     │  │     (.pkl)      │    │
│  └─────────────┘  └──────────────┘  └─────────────────┘    │
│                                                             │
│                    ⬇️ PACKAGING ⬇️                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                🐳 CONTAINER                            ││
│  │                                                        ││
│  │  ┌─────────────┐    ┌─────────────┐    ┌──────────┐   ││
│  │  │   FastAPI   │    │    Model    │    │   Data   │   ││
│  │  │ (main.py)   │ ←→ │ (inference) │ ←→ │ (models) │   ││
│  │  └─────────────┘    └─────────────┘    └──────────┘   ││
│  │                                                        ││
│  │  Port 8000 → 🌐 REST API Endpoints                    ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 🎯 Preparación del Modelo {#preparacion}

### **1. Estructura de Proyecto**

```
house-price-predictor/
├── 📁 src/
│   └── 📁 api/
│       ├── main.py           # FastAPI app
│       ├── inference.py      # Lógica de predicción
│       ├── schemas.py        # Modelos Pydantic
│       ├── requirements.txt  # Dependencias API
│       └── __init__.py
├── 📁 models/
│   └── 📁 trained/
│       ├── house_price_model.pkl      # Modelo entrenado
│       └── preprocessor.pkl           # Pipeline de preprocessing
├── 📁 data/
│   └── 📁 processed/
│       └── featured_house_data.csv    # Datos procesados
├── Dockerfile                # Configuración del contenedor
└── requirements.txt          # Dependencias del proyecto
```

### **2. Archivos del Modelo**

Los archivos esenciales para el modelo:

```python
# Generados durante el entrenamiento:
models/trained/house_price_model.pkl      # Modelo ML (RandomForest, etc.)
models/trained/preprocessor.pkl           # Pipeline de transformación
```

---

## 🚀 Desarrollo de la API FastAPI {#fastapi}

### **1. Esquemas de Datos (`src/api/schemas.py`)**

```python
from pydantic import BaseModel, Field
from typing import List, Literal

class HousePredictionRequest(BaseModel):
    """Schema para request de predicción de precios de casas"""
    sqft: float = Field(..., gt=1000, lt=5000, description="Square footage of the house")
    bedrooms: int = Field(..., ge=1, le=6, description="Number of bedrooms")
    bathrooms: float = Field(..., gt=0.5, le=5.0, description="Number of bathrooms")
    location: Literal["Rural", "Suburb", "Urban", "Downtown", "Waterfront", "Mountain"]
    year_built: int = Field(..., ge=1945, le=2023, description="Year the house was built")
    condition: Literal["Poor", "Fair", "Good", "Excellent"]
    price_per_sqft: float = Field(..., gt=50, lt=1000, description="Expected price per square foot")

    class Config:
        schema_extra = {
            "example": {
                "sqft": 1527,
                "bedrooms": 2,
                "bathrooms": 1.5,
                "location": "Suburb",
                "year_built": 1956,
                "condition": "Good",
                "price_per_sqft": 320
            }
        }

class PredictionResponse(BaseModel):
    """Schema para response de predicción"""
    predicted_price: float = Field(..., description="Predicted house price in dollars")
    confidence_interval: List[float] = Field(..., description="90% confidence interval")
    features_importance: dict = Field(default={}, description="Feature importance scores")
    prediction_time: str = Field(..., description="Timestamp of the prediction")
```

### **2. Lógica de Inferencia (`src/api/inference.py`)**

```python
import joblib
import pandas as pd
import os
from datetime import datetime
from pathlib import Path
from schemas import HousePredictionRequest, PredictionResponse

# Configuración de rutas para Docker
MODEL_PATH = "models/trained/house_price_model.pkl"
PREPROCESSOR_PATH = "models/trained/preprocessor.pkl"

try:
    # Cargar modelo y preprocessor al inicio
    model = joblib.load(MODEL_PATH)
    preprocessor = joblib.load(PREPROCESSOR_PATH)
    print(f"✅ Model loaded: {type(model).__name__}")
    print(f"✅ Preprocessor loaded successfully")
except Exception as e:
    raise RuntimeError(f"Error loading model or preprocessor: {str(e)}")

def predict_price(request: HousePredictionRequest) -> PredictionResponse:
    """
    Realiza predicción de precio de casa basada en características.

    Args:
        request: Datos de la casa (sqft, bedrooms, etc.)

    Returns:
        PredictionResponse: Precio predicho y metadatos
    """
    try:
        # 1. Preparar datos de entrada
        input_data = pd.DataFrame([{
            'sqft': request.sqft,
            'bedrooms': request.bedrooms,
            'bathrooms': request.bathrooms,
            'location': request.location,
            'year_built': request.year_built,
            'condition': request.condition,
            'price_per_sqft': request.price_per_sqft
        }])

        # 2. Calcular features derivadas
        current_year = datetime.now().year
        input_data['house_age'] = current_year - input_data['year_built']
        input_data['bed_bath_ratio'] = input_data['bedrooms'] / input_data['bathrooms']

        # 3. Aplicar preprocessing
        processed_features = preprocessor.transform(input_data)

        # 4. Hacer predicción
        predicted_price = model.predict(processed_features)[0]
        predicted_price = round(float(predicted_price), 2)

        # 5. Calcular intervalo de confianza
        confidence_interval = [
            round(predicted_price * 0.9, 2),
            round(predicted_price * 1.1, 2)
        ]

        return PredictionResponse(
            predicted_price=predicted_price,
            confidence_interval=confidence_interval,
            features_importance={},
            prediction_time=datetime.now().isoformat()
        )

    except Exception as e:
        raise Exception(f"Prediction failed: {str(e)}")

def batch_predict(requests: list[HousePredictionRequest]) -> list[PredictionResponse]:
    """Predicción en lotes"""
    return [predict_price(req) for req in requests]
```

### **3. Aplicación FastAPI (`src/api/main.py`)**

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from inference import predict_price, batch_predict
from schemas import HousePredictionRequest, PredictionResponse

# Inicializar FastAPI
app = FastAPI(
    title="🏠 House Price Prediction API",
    description="API para predecir precios de casas usando Machine Learning",
    version="1.0.0",
    contact={
        "name": "MLOps Team",
        "url": "https://github.com/tu-usuario/house-price-predictor",
    },
)

# Configurar CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
async def root():
    """Endpoint raíz"""
    return {"message": "🏠 House Price Prediction API", "status": "running"}

@app.get("/health")
async def health_check():
    """Health check para monitoreo"""
    return {"status": "healthy", "model_loaded": True}

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: HousePredictionRequest):
    """
    Predecir precio de una casa individual

    - **sqft**: Tamaño en pies cuadrados
    - **bedrooms**: Número de habitaciones
    - **bathrooms**: Número de baños
    - **location**: Ubicación (Rural, Suburb, Urban, Downtown, Waterfront, Mountain)
    - **year_built**: Año de construcción
    - **condition**: Condición (Poor, Fair, Good, Excellent)
    - **price_per_sqft**: Precio estimado por pie cuadrado en el área
    """
    try:
        result = predict_price(request)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/batch-predict")
async def batch_predict_endpoint(requests: list[HousePredictionRequest]):
    """Predicción en lotes para múltiples casas"""
    try:
        results = batch_predict(requests)
        return {"predictions": results, "count": len(results)}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### **4. Dependencias (`src/api/requirements.txt`)**

```txt
fastapi==0.116.1
uvicorn==0.35.0
pydantic==2.8.2
pandas==2.2.3
scikit-learn==1.7.1
joblib==1.4.2
numpy==1.26.0
python-multipart==0.0.17
```

---

## 🐳 Dockerización {#dockerizacion}

### **1. Dockerfile**

```dockerfile
# Imagen base ligera de Python
FROM python:3.11-slim

# Establecer directorio de trabajo
WORKDIR /app

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copiar y instalar dependencias Python
COPY src/api/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la API
COPY src/api/main.py .
COPY src/api/inference.py .
COPY src/api/schemas.py .
COPY src/api/__init__.py .

# Crear directorio y copiar modelos entrenados
RUN mkdir -p models/trained
COPY models/trained/ ./models/trained/

# Crear usuario no-root por seguridad
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Exponer puerto
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Comando de inicio
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### **2. Construir Imagen con Versionado**

```bash
# Construir la imagen Docker con versión específica
docker build -t house-price-model:v1.0.0 .
docker build -t house-price-model:latest .

# Verificar la imagen
docker images | grep house-price-model
```

### **3. Ejecutar Contenedor**

```bash
# Ejecutar en modo detached con versión específica
docker run -d -p 8000:8000 --name house-price-api house-price-model:v1.0.0

# Verificar que está corriendo
docker ps

# Ver logs
docker logs house-price-api
```

---

## 🧪 Testing y Validación {#testing}

### **1. Health Check**

```bash
# Verificar que la API está funcionando
curl http://localhost:8000/health

# Respuesta esperada:
# {"status": "healthy", "model_loaded": true}
```

### **2. Documentación Automática**

Visita en tu navegador:

- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

### **3. Test de Predicción Individual**

```bash
curl -X POST "http://localhost:8000/predict" \
-H "Content-Type: application/json" \
-d '{
  "sqft": 1527,
  "bedrooms": 2,
  "bathrooms": 1.5,
  "location": "Suburb",
  "year_built": 1956,
  "condition": "Good",
  "price_per_sqft": 320
}'
```

**Respuesta esperada:**

```json
{
  "predicted_price": 566959.08,
  "confidence_interval": [510263.17, 623654.99],
  "features_importance": {},
  "prediction_time": "2025-07-24T18:25:24.244040"
}
```

### **4. Test de Predicción en Lotes**

```bash
curl -X POST "http://localhost:8000/batch-predict" \
-H "Content-Type: application/json" \
-d '[
  {
    "sqft": 1527,
    "bedrooms": 2,
    "bathrooms": 1.5,
    "location": "Suburb",
    "year_built": 1956,
    "condition": "Good",
    "price_per_sqft": 320
  },
  {
    "sqft": 2100,
    "bedrooms": 3,
    "bathrooms": 2.0,
    "location": "Urban",
    "year_built": 2005,
    "condition": "Excellent",
    "price_per_sqft": 280
  }
]'
```

### **5. Scripts de Testing**

```python
# test_api.py
import requests
import json

API_URL = "http://localhost:8000"

def test_health():
    response = requests.get(f"{API_URL}/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"
    print("✅ Health check passed")

def test_prediction():
    payload = {
        "sqft": 1527,
        "bedrooms": 2,
        "bathrooms": 1.5,
        "location": "Suburb",
        "year_built": 1956,
        "condition": "Good",
        "price_per_sqft": 320
    }

    response = requests.post(f"{API_URL}/predict", json=payload)
    assert response.status_code == 200

    result = response.json()
    assert "predicted_price" in result
    assert result["predicted_price"] > 0
    print(f"✅ Prediction test passed: ${result['predicted_price']:,.2f}")

if __name__ == "__main__":
    test_health()
    test_prediction()
    print("🎉 All tests passed!")
```

---

---

## 📦 Publicación en Docker Hub

### **1. Preparación para Publicar**

```bash
# 1. Verificar que la imagen funciona localmente
docker run -d -p 8000:8000 --name test-api house-price-model:v1.0.0
curl http://localhost:8000/health
docker stop test-api && docker rm test-api
```

### **2. Login y Tag de Imagen**

```bash
# Login a Docker Hub
docker login
# Introduce tu username y password de Docker Hub

# Tag de la imagen con tu usuario de Docker Hub
docker tag house-price-model:v1.0.0 tu-usuario/house-price-model:v1.0.0
docker tag house-price-model:latest tu-usuario/house-price-model:latest

# Ejemplo real:
# docker tag house-price-model:v1.0.0 pytuxi/house-price-model:v1.0.0
# docker tag house-price-model:latest pytuxi/house-price-model:latest
```

### **3. Publicar en Docker Hub**

```bash
# Push de la versión específica (recomendado)
docker push tu-usuario/house-price-model:v1.0.0

# Push de latest (opcional)
docker push tu-usuario/house-price-model:latest

# Verificar en Docker Hub
echo "✅ Imagen disponible en: https://hub.docker.com/r/tu-usuario/house-price-model"
```

---

## 📥 Descarga y Uso de la Imagen

### **1. Descargar desde Docker Hub**

```bash
# Descargar versión específica (recomendado)
docker pull tu-usuario/house-price-model:v1.0.0

# O descargar latest (puede tener cambios no deseados)
docker pull tu-usuario/house-price-model:latest
```

### **2. Ejecutar Imagen Descargada**

```bash
# Ejecutar la versión específica descargada
docker run -d -p 8000:8000 --name house-price-api tu-usuario/house-price-model:v1.0.0

# Para Apple Silicon (ARM64), especificar plataforma si es necesario
docker run -d -p 8000:8000 --platform linux/arm64 --name house-price-api tu-usuario/house-price-model:v1.0.0

# Verificar que está corriendo
docker ps
```

### **3. Verificación Post-Descarga**

```bash
# Health check
curl http://localhost:8000/health

# Test de predicción
curl -X POST "http://localhost:8000/predict" \
-H "Content-Type: application/json" \
-d '{
  "sqft": 1527,
  "bedrooms": 2,
  "bathrooms": 1.5,
  "location": "Suburb",
  "year_built": 1956,
  "condition": "Good",
  "price_per_sqft": 320
}'

# Abrir documentación en navegador
open http://localhost:8000/docs
```

---

## 📝 Comandos de Referencia Rápida

```bash
# Desarrollo Local
python -m uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --reload

# Docker Build & Run (Local)
docker build -t house-price-model:v1.0.0 .
docker run -d -p 8000:8000 --name api house-price-model:v1.0.0

# Publicar en Docker Hub
docker tag house-price-model:v1.0.0 tu-usuario/house-price-model:v1.0.0
docker push tu-usuario/house-price-model:v1.0.0

# Descargar y usar desde Docker Hub
docker pull tu-usuario/house-price-model:v1.0.0
docker run -d -p 8000:8000 --name api tu-usuario/house-price-model:v1.0.0

# Testing
curl http://localhost:8000/health
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d @test_data.json

# Logs & Debug
docker logs api
docker exec -it api /bin/bash

# Cleanup
docker stop api && docker rm api
docker rmi tu-usuario/house-price-model:v1.0.0
```

---

## 🏷️ Gestión de Versiones

### **Estrategia de Versionado**

```bash
# Versiones semánticas
docker build -t house-price-model:v1.0.0 .  # Release inicial
docker build -t house-price-model:v1.0.1 .  # Bug fix
docker build -t house-price-model:v1.1.0 .  # Nueva funcionalidad
docker build -t house-price-model:v2.0.0 .  # Cambio mayor

# Tags por ambiente
docker tag house-price-model:v1.0.0 tu-usuario/house-price-model:dev
docker tag house-price-model:v1.0.0 tu-usuario/house-price-model:staging
docker tag house-price-model:v1.0.0 tu-usuario/house-price-model:prod

# Tag por fecha
docker tag house-price-model:v1.0.0 tu-usuario/house-price-model:2025-07-24
```

### **Mejores Prácticas**

1. **✅ Siempre usar versiones específicas** en producción
2. **✅ Testear localmente** antes de publicar
3. **✅ Usar tags semánticos** (v1.0.0, v1.1.0, etc.)
4. **✅ Mantener latest** actualizado pero no usarlo en producción
5. **✅ Documentar cambios** entre versiones

### **Ejemplo Real de Uso**

```bash
# Construir nueva versión
docker build -t house-price-model:v1.0.1 .

# Test local
docker run -d -p 8000:8000 --name test house-price-model:v1.0.1
curl http://localhost:8000/health
docker stop test && docker rm test

# Publicar
docker tag house-price-model:v1.0.1 pytuxi/house-price-model:v1.0.1
docker push pytuxi/house-price-model:v1.0.1

# Uso en otros servidores
docker pull pytuxi/house-price-model:v1.0.1
docker run -d -p 8000:8000 pytuxi/house-price-model:v1.0.1
```

---

## 🎯 Próximos Pasos

1. **CI/CD Pipeline**: Automatizar builds y despliegues
2. **Kubernetes**: Orquestar contenedores en producción
3. **API Gateway**: Implementar rate limiting y autenticación
4. **Model Versioning**: MLflow para gestión de versiones
5. **A/B Testing**: Comparar modelos en producción

---

## 📚 Recursos Adicionales

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [MLOps with Docker](https://neptune.ai/blog/mlops-with-docker)
- [Prometheus Monitoring](https://prometheus.io/docs/guides/go-application/)

---

**🏆 ¡Felicidades! Has dockerizado exitosamente tu modelo ML con FastAPI. Tu modelo ahora está listo para producción, es escalable y fácil de desplegar en cualquier entorno.**
