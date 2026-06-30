# 🛵 SpeedyEats: Otimização Logística e Redução de Churn Operacional

Este repositório contém a documentação técnica, os scripts de Engenharia de Dados (Python) e as consultas de negócio (SQL) utilizadas no desenvolvimento do projeto SpeedyEats.

> 📊 **Acesse a documentação completa com o Dashboard interativo aqui:** https://app.notion.com/p/SpeedyEats-Otimiza-o-Log-stica-e-Redu-o-de-Churn-Operacional-38ee7ba9623380a68e20e0637fdb0b59?source=copy_link

---

## 🏗️ 1. Engenharia e Simulação de Dados (Python)

Para este projeto, foi desenvolvido um script em Python (utilizando Pandas e Numpy) para simular uma base histórica realista de **5.000 pedidos** contendo gargalos operacionais intencionais (atrasos em horários de pico e cancelamentos por fatores climáticos).

Também foi implementado um modelo de Machine Learning (**Random Forest Classifier**) para calcular a importância das variáveis no estouro do SLA de 45 minutos.

<details>
<summary><b>📂 Clique aqui para visualizar o Código Python (Google Colab)</b></summary>

```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

# Configuração de semente para reproducibilidade
np.random.seed(42)
n_orders = 5000

# Simulação dos dados base
categories = ['Burgers', 'Pizza', 'Sushi', 'Brasileira']
statuses = ['Entregue', 'Cancelado']

# Geração de datas aleatórias distribuídas em um mês
start_date = datetime(2026, 6, 1)
dates = [start_date + timedelta(seconds=int(np.random.randint(0, 30*24*3600))) for _ in range(n_orders)]

data = pd.DataFrame({
    'order_id': range(1, n_orders + 1),
    'restaurant_category': np.random.choice(categories, n_orders, p=[0.3, 0.3, 0.2, 0.2]),
    'weather_condition': np.random.choice(['Limpo', 'Chuva'], n_orders, p=[0.8, 0.2]),
    'order_timestamp': dates,
    'order_amount_brl': np.random.uniform(30, 150, n_orders).round(2)
})

data['hour'] = data['order_timestamp'].dt.hour
data['is_weekend'] = data['order_timestamp'].dt.dayofweek.isin([5, 6]).astype(int)

# Aplicação dos gargalos operacionais na simulação
prep_time = []
driver_time = []
travel_time = []
status_list = []
reason_list = []

for idx, row in data.iterrows():
    p = np.random.normal(20, 5)
    d = np.random.normal(5, 2)
    t = np.random.normal(15, 4)
    
    # Gargalo 1: Hamburguerias em horários de pico no final de semana
    if row['restaurant_category'] == 'Burgers' and row['is_weekend'] == 1 and (18 <= row['hour'] <= 22):
        p += np.random.normal(15, 3)
        
    # Gargalo 2: Impacto severo de chuva na logística de rua
    if row['weather_condition'] == 'Chuva':
        t += np.random.normal(10, 3)
        if np.random.rand() < 0.14:  # 14% de cancelamento na chuva
            status_list.append('Cancelado')
            reason_list.append('Falta de Entregador' if np.random.rand() < 0.7 else 'Atraso Excessivo')
            prep_time.append(np.nan)
            driver_time.append(np.nan)
            travel_time.append(np.nan)
            continue
    else:
        if np.random.rand() < 0.02:  # 2% de cancelamento em clima limpo
            status_list.append('Cancelado')
            reason_list.append('Arrependimento do Cliente')
            prep_time.append(np.nan)
            driver_time.append(np.nan)
            travel_time.append(np.nan)
            continue
            
    status_list.append('Entregue')
    reason_list.append(np.nan)
    prep_time.append(max(5, round(p, 1)))
    driver_time.append(max(1, round(d, 1)))
    travel_time.append(max(5, round(t, 1)))

data['prep_time_min'] = prep_time
data['driver_wait_time_min'] = driver_time
data['travel_time_min'] = travel_time
data['total_time_min'] = data['prep_time_min'] + data['driver_wait_time_min'] + data['travel_time_min']
data['order_status'] = status_list
data['cancellation_reason'] = reason_list

# Exportação da base tratada para o BigQuery
data.to_csv('dados_delivery_speedyeats.csv', index=False)

# Modelagem Preditiva com Machine Learning (Apenas pedidos concluídos)
df_ml = data[data['order_status'] == 'Entregue'].copy()
df_ml['is_late'] = (df_ml['total_time_min'] > 45).astype(int)
df_ml = pd.get_dummies(df_ml, columns=['restaurant_category', 'weather_condition'], drop_first=False)

features = ['hour', 'is_weekend', 'restaurant_category_Burgers', 'weather_condition_Chuva']
X = df_ml[features]
y = df_ml['is_late']

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
model = RandomForestClassifier(random_state=42)
model.fit(X_train, y_train)

print("\n🤖 RESULTADOS DA IA:")
importances = model.feature_importances_
for feature, imp in zip(features, importances):
    nome_pt = {
        'hour': 'Hora do dia',
        'is_weekend': 'Ser fim de semana',
        'restaurant_category_Burgers': 'Categoria ser Hamburgueria (Burgers)',
        'weather_condition_Chuva': 'Condição climática ser Chuva'
    }[feature]
    print(f"-> Impacto do fator '{nome_pt}' no atraso do pedido: {imp*100:.2f}%")
'''
</details>

---

## 🔍 2. Consultas de Negócio (SQL - Google BigQuery)
Os dados simulados foram estruturados e carregados no Google BigQuery para a extração de indicadores operacionais e financeiros estratégicos.

<details>
<summary><b>📂 Clique aqui para visualizar o Código SQL (Google BigQuery)</b></summary>

Query 1: Impacto Financeiro dos Cancelamentos por Clima

```sql
SELECT 
    weather_condition AS clima,
    cancellation_reason AS motivo_cancelamento,
    COUNT(order_id) AS total_pedidos_cancelados,
    ROUND(SUM(order_amount_brl), 2) AS faturamento_perdido_reais
FROM 
    `portfolio-bi-500921.speedyeats_operacoes.pedidos`
WHERE 
    order_status = 'Cancelado'
GROUP BY 
    clima, motivo_cancelamento
ORDER BY 
    faturamento_perdido_reais DESC;
"""
