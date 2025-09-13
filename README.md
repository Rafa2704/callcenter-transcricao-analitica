# 🎧 callcenter-transcricao-analitica

Projeto que automatiza a extração de dados relevantes a partir de transcrições de chamadas telefônicas usando **serviços gerenciados da AWS**. A solução permite identificar automaticamente **CPF informado**, **número de protocolo** e **motivo da ligação**, viabilizando análises operacionais e inteligência de atendimento.

---

## 🚦 Visão Geral do Processo

1. 🎙️ **Carga de áudios** é enviada para um bucket S3 (ex: `s3://callcenter-transcricoes/raw/`).
2. 🧠 O **Amazon Transcribe** processa os áudios e gera arquivos `.json` com as transcrições estruturadas.
3. ☁️ Esses arquivos `.json` são salvos em uma pasta no S3 (ex: `s3://callcenter-transcricoes/json/`).
4. 🔍 Um **AWS Glue Crawler** escaneia os arquivos e cria uma tabela no **Glue Data Catalog**, disponibilizando os dados para consulta no **Amazon Athena**.
5. 📊 A partir do Athena, executamos queries SQL que:
   - Extraem o **CPF informado**
   - Capturam o **número de protocolo**
   - Classificam o **motivo da ligação** com base em palavras-chave

Opcionalmente, é possível incluir um **AWS Glue Job em PySpark** para processamentos adicionais ou persistência em formatos otimizados como Parquet.

---

## 🧱 Arquitetura

```
Usuário → Áudio → [S3 Bucket: raw/] → Transcribe → [S3 Bucket: json/] 
→ Glue Crawler → Glue Data Catalog → Athena / Glue Job
```

---

## 🧠 Exemplo de Transcrição

```text
"Oi. Você ligou para o serviço de atendimento ao Consumidor SAC da CNP Seguradora. (...)
Eu quero cancelar meu título de capitalização. Certo. A senhora me confirma por favor o seu CPF. 016.061.03547. (...)
A sua ligação gerou o protocolo 250848252064."
```

---

## 🧪 Query SQL principal (Athena)

```sql
WITH transcricoes AS (
  SELECT
    jobname,
    accountid,
    r.transcript,
    
    -- Extrai CPF após a palavra "CPF"
    REGEXP_EXTRACT(r.transcript, '(?i)(?:cpf)\D*(\d{11})') AS cpf_extraido,

    -- Extrai número de protocolo (12 dígitos)
    REGEXP_EXTRACT(r.transcript, '\b(\d{12})\b') AS protocolo,

    -- Motivo do contato (via palavras-chave)
    CASE
      WHEN r.transcript LIKE '%cancelar meu título%' THEN 'Cancelar título de capitalização'
      WHEN r.transcript LIKE '%consórcio%' THEN 'Consórcio'
      WHEN r.transcript LIKE '%plano odontológico%' THEN 'Plano odontológico'
      ELSE 'Outro'
    END AS motivo_contato

  FROM callcenter_db.transcricao_transcricoes
  CROSS JOIN UNNEST(results.transcripts) AS t(r)
)

SELECT * FROM transcricoes;
```

---



## 🧾 Requisitos (para execução local)

```txt
pyspark==3.1.2
boto3
```

---

## 📊 Exemplo de Resultado

| jobname                            | cpf_extraido | protocolo     | motivo_contato                    |
|------------------------------------|--------------|---------------|-----------------------------------|
| transcricao-9d9e9826-xxxxxxx       | 01606103547  | 250848252064  | Cancelar título de capitalização |

---

## ✅ Próximos passos

- [ ] Enriquecer o motivo com IA generativa ou embeddings
- [ ] Classificação automática dos atendimentos
- [ ] Dashboard (QuickSight, Looker Studio, etc)
- [ ] Alertas em tempo real com EventBridge ou Lambda

---

## 🧑‍💻 Autor

Rafael Santos  
Especialista em Engenharia de Dados  
GitHub: (https://github.com/Rafa2704)  
LinkedIn: [linkedin.com/in/rafaelcarlossantos](linkedin.com/in/rafaelcarlossantos)
