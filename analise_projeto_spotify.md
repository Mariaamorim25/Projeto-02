## 5.1 Processar e preparar a base de dados

===============================================================
5.1.1 Conectar/importar dados para outras ferramentas
===============================================================

Nesta etapa foi realizada a importação dos arquivos para o BigQuery - track_in_competition - competition, track_in_spotify - spotify e track_technical_info - technical_info.

===============================================================
5.1.2 Identificar e tratar valores nulos
===============================================================

--Identificação dos valores nulos
```
#track_technical_info
SELECT
COUNT(*) AS total_linhas,
COUNTIF(track_id IS NULL) AS track_id_nulos,
COUNTIF(bpm IS NULL) AS bpm_nulos,
COUNTIF(key IS NULL) AS key_nulos,
COUNTIF(mode IS NULL) AS mode_nulos,
COUNTIF(danceability_ IS NULL) AS danceability_nulos,
COUNTIF(valence_ IS NULL) AS valence_nulos,
COUNTIF(energy_ IS NULL) AS energy_nulos,
COUNTIF(acousticness_ IS NULL) AS acousticness_nulos,
COUNTIF(instrumentalness_ IS NULL) AS instrumentalness_nulos,
COUNTIF(liveness_ IS NULL) AS liveness_nulos,
COUNTIF(instrumentalness_ IS NULL) AS instrumentalness_nulos,
COUNTIF(speechiness_ IS NULL) AS speechiness_nulos,
FROM `projeto-02-hipoteses-456611.Spotify.track_technical_info`;
```

-- Atualização valores nulos na coluna KEY como "Desconhecido" e criar uma nova view
```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_technical_info_limpa` AS
SELECT
  track_id,
  bpm,
  IFNULL(key, 'Desconhecido') AS key,
  mode,
  danceability_,
  valence_,
  energy_,
  acousticness_,
  instrumentalness_,
  liveness_,
  speechiness_
FROM `projeto-02-hipoteses-456611.Spotify.track_technical_info`;
```

-- Atualização os valores nulos na coluna IN_SHAZAM_CHARTS como 0 e criar uma nova view
```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.trackincompetition_limpa` AS
SELECT
  track_id,
  IFNULL(inappleplaylists, 0) AS inappleplaylists,
  IFNULL(inapplecharts, 0) AS inapplecharts,
  IFNULL(indeezerplaylists, 0) AS indeezerplaylists,
  IFNULL(indeezercharts, 0) AS indeezercharts,
  IFNULL(inshazamcharts, 0) AS inshazamcharts
FROM 
  `projeto-02-hipoteses-456611.Spotify.trackincompetition`;
```

===============================================================
5.1.3 Identificar e tratar valores duplicados
===============================================================

-- Identificação dos valores duplicados realizando uma concatenação do nome do artista e nome da música
```
SELECT 
  CONCAT(track_name, ' - ', artists_name) AS musica_artista,
  COUNT(*) AS total_ocorrencias
FROM 
  `projeto-02-hipoteses-456611.Spotify.track_in_spotify`
GROUP BY 
  musica_artista
HAVING 
  COUNT(*) > 1;

-- Remoção da segunda duplicada por ordem de ID  
```
  CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_in_spotify_sem_duplicatas` AS
WITH numeradas AS (
  SELECT 
    *,
    ROW_NUMBER() OVER (
      PARTITION BY track_name, artist_name 
      ORDER BY track_id 
    ) AS linha
  FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify`
)
SELECT *
FROM numeradas
WHERE linha = 1;
```

===============================================================
5.1.4 Identificar e tratar dados fora do escopo de análise
===============================================================

-- Colunas KEY e MODE removidas da base 
```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_technical_info_limpa` AS
SELECT 
  * EXCEPT(key, mode)
FROM 
  `projeto-02-hipoteses-456611.Spotify.track_technical_info`;
```

===============================================================
5.1.5 Identificar e tratar dados discrepantes em variáveis categóricas
===============================================================

-- Dados com caracteres discrepantes removidos
```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_in_spotify_sem_duplicatas` AS
SELECT
  track_id,
  REGEXP_REPLACE(track_name, r'�', '') AS track_name,
  REGEXP_REPLACE(artists_name, r'�', '') AS artists_name,
  artist_count,
  released_year,
  released_month,
  released_day,
  in_spotify_playlists,
  in_spotify_charts,
  streams
FROM
  `projeto-02-hipoteses-456611.Spotify.track_in_spotify`;
```

===============================================================
5.1.6 Identificar e tratar dados discrepantes em variáveis numéricas
===============================================================

-- Identificação dos dados discrepantes
```
SELECT
  MIN(streams) AS min_streams,
  MAX(streams) AS max_streams,
  AVG(streams) AS media_streams,
  STDDEV(streams) AS desvio_padrao_streams
FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify_sem_duplicatas`;
```

-- Remoção da linha com dado discrepante 
```
SELECT
  track_id,
  track_name,
  artists_name,
  streams
FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify_sem_duplicatas`
WHERE streams > 1751714589
ORDER BY streams DESC;
```

===============================================================
5.1.7 Verificar e alterar o tipo de dados
===============================================================

```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_in_spotify_limpa_final` AS
SELECT
  track_id,
  REGEXP_REPLACE(track_name, r'�', '') AS track_name,
  REGEXP_REPLACE(artists_name, r'�', '') AS artists_name,
  artist_count,
  released_year,
  released_month,
  released_day,
  in_spotify_playlists,
  in_spotify_charts,
  CAST(streams AS INT64) AS streams
FROM
  `projeto-02-hipoteses-456611.Spotify.track_in_spotify`
WHERE 
  SAFE_CAST(streams AS INT64) IS NOT NULL
  AND track_id NOT IN ('4061483', '0:00'); 
```

===============================================================
5.1.8 Criar novas variáveis 
===============================================================

-- Criação das novas variaveis data_lancamento e total_charts
```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_in_spotify_com_novas_variaveis` AS
SELECT
  *,
  DATE(released_year, released_month, released_day) AS data_lancamento

  CASE 
    WHEN streams < 51143742 THEN 'baixo_sucesso'
    WHEN streams BETWEEN 51143742 AND 618000000 THEN 'sucesso_medio'
    WHEN streams > 618000000 THEN 'alto_sucesso'
    ELSE 'indefinido'
  END AS faixa_de_sucesso

FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify_limpa_final`;
```

```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.track_in_competition_com_variaveis` AS
SELECT
  *,
  
  (IFNULL(in_apple_charts, 0) + IFNULL(in_deezer_charts, 0) + IFNULL(in_shazam_charts, 0)) AS total_charts,

  CASE
    WHEN (IFNULL(in_apple_charts, 0) + IFNULL(in_deezer_charts, 0) + IFNULL(in_shazam_charts, 0)) = 0 THEN 'nao_ranqueado'
    WHEN (IFNULL(in_apple_charts, 0) + IFNULL(in_deezer_charts, 0) + IFNULL(in_shazam_charts, 0)) <= 5 THEN 'baixa'
    WHEN (IFNULL(in_apple_charts, 0) + IFNULL(in_deezer_charts, 0) + IFNULL(in_shazam_charts, 0)) <= 15 THEN 'media'
    ELSE 'alta'
  END AS faixa_de_popularidade

FROM `projeto-02-hipoteses-456611.Spotify.trackincompetition_escopo_filtrado`;
```
===============================================================
5.1.9 Unir tabelas
===============================================================

-- União das tabelas tratadas 

```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.analise_final_musicas` AS
SELECT 
  s.track_id,
  s.track_name,
  s.artists_name,
  s.artist_count,
  s.streams,
  s.data_lancamento,
  s.faixa_de_sucesso,
  s.in_spotify_playlists,
  s.in_spotify_charts,

  c.total_playlists,
  c.total_charts,
  c.faixa_de_popularidade,

  t.perfil_energia,
  t.eh_instrumental,
  t.faixa_bpm

FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify_com_novas_variaveis` s
LEFT JOIN `projeto-02-hipoteses-456611.Spotify.track_in_competition_com_variaveis` c
  ON s.track_id = c.track_id
LEFT JOIN `projeto-02-hipoteses-456611.Spotify.track_technical_info_com_variaveis` t
  ON s.track_id = t.track_id;
```

===============================================================
5.1.10 Construir tabelas auxiliares
===============================================================

```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.analise_final_com_musicas_solo` AS

WITH artistas_solo AS (
  SELECT
    artists_name,
    COUNT(*) AS total_musicas_solo
  FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify_com_novas_variaveis`
  WHERE artist_count = 1
  GROUP BY artists_name
)

SELECT
  a.*,
  IFNULL(s.total_musicas_solo, 0) AS total_musicas_solo
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_musicas` a
LEFT JOIN artistas_solo s
  ON a.artists_name = s.artists_name;
```

## 5.2 Fazer uma análise exploratória

===============================================================
5.2.1 Agrupar dados de acordo com variáveis categóricas
===============================================================

```
SELECT
  artists_name,
  COUNT(*) AS total_musicas,
  SUM(streams) AS total_streams
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_musicas`
GROUP BY artists_name
ORDER BY total_musicas DESC;

SELECT
  EXTRACT(YEAR FROM data_lancamento) AS ano_lancamento,
  COUNT(*) AS total_musicas,
  SUM(streams) AS total_streams
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_musicas`
GROUP BY ano_lancamento
ORDER BY ano_lancamento;

SELECT
  faixa_de_sucesso,
  COUNT(*) AS total_musicas,
  AVG(streams) AS media_streams
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_musicas`
GROUP BY faixa_de_sucesso
ORDER BY total_musicas DESC;
```

5.2.2 Ver variáveis categóricas

![grafico01](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/grafico01.png)

5.2.3 Aplicar medidas de tendência central

![tendencia_central](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/tendencia_central.png)

5.2.4 Ver distribuição

-- Codigo Python no Power BI para criar o histograma

```
import matplotlib.pyplot as plt
import pandas as pd

# Pega os dados da tabela importada pelo Power BI
data = dataset[['streams']].dropna()

# Cria o histograma com 10 faixas (bins)
plt.figure(figsize=(10, 6))
plt.hist(data['streams'], bins=10, color='skyblue', edgecolor='black', alpha=0.7)

# Configurações do gráfico
plt.xlabel('Streams (Quantidade de execuções)')
plt.ylabel('Número de músicas')
plt.title('Distribuição de Streams das Músicas')
plt.grid(axis='y', linestyle='--', alpha=0.5)
plt.tight_layout()

# Mostra o gráfico no visual do Power BI
plt.show()
```
![histograma](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/histograma.png)

5.2.5 Aplicar medidas de dispersão

![desvio_padrao](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/desvio_padrao.png)

5.2.6 Visualizar o comportamento dos dados ao longo do tempo

![longo_dos_anos](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/longo_dos_anos.png)

5.2.7 Calcular quartis, decis ou percentis

```
SELECT
  PERCENTILE_CONT(streams, 0.25) OVER() AS Q1,
  PERCENTILE_CONT(streams, 0.50) OVER() AS Q2,
  PERCENTILE_CONT(streams, 0.75) OVER() AS Q3
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_com_musicas_solo`
LIMIT 1;
```
```
CREATE OR REPLACE VIEW `projeto-02-hipoteses-456611.Spotify.analise_final_quartis` AS
SELECT
  *,
  CASE
    WHEN streams <= 20000000 THEN 'Q1 - Baixo sucesso'
    WHEN streams <= 51000000 THEN 'Q2 - Sucesso médio-baixo'
    WHEN streams <= 120000000 THEN 'Q3 - Sucesso médio-alto'
    ELSE 'Q4 - Alto sucesso'
  END AS faixa_quartil_streams
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_com_musicas_solo`;
```
![quartis](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/quartis.png)

5.2.8 Calcular correlação entre variáveis

```
SELECT
  CORR(streams, total_playlists) AS correlacao_streams_playlists
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_quartis`;
```
-- Correlação 0.6329 - 	Moderada a forte

## 5.3 Aplicar técnica de análise

5.3.1 Aplicar segmentação

![segmentacao](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/segmentacao.png)

5.3.2 Validar hipótese

#### 1. Músicas com BPM mais altos fazem mais sucesso em streams no Spotify

![hipotese01](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese01.png)

#### 2. As músicas mais populares no ranking do Spotify também possuem um comportamento semelhante em outras plataformas como Deezer.

![hipotese02](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese02.png)

#### 3. A presença de uma música em um maior número de playlists está relacionada a um maior número de streams.

![hipotese03](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese03.png)

#### 4. Artistas com maior número de músicas no Spotify têm mais streams.

![hipotese04](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese04.png)

#### 5. As características da música influenciam no sucesso em termos de streams no Spotify.

![hipotese05](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese05.png)


## 5.4 Resumir informações em um dashboard ou relatório

![relatorio_geral](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/relatorio_geral.png)





















