## 5.1 Processar e preparar a base de dados

===============================================================
5.1.1 Conectar/importar dados para outras ferramentas
===============================================================

Nesta etapa foi realizada a importação dos arquivos para o BigQuery - track_in_competition - competition, track_in_spotify - spotify e track_technical_info - technical_info.

===============================================================
5.1.2 Identificar e tratar valores nulos
===============================================================

--Identificação dos valores nulos
```sql
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
```sql
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
```sql
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
```sql
SELECT 
  CONCAT(track_name, ' - ', artists_name) AS musica_artista,
  COUNT(*) AS total_ocorrencias
FROM 
  `projeto-02-hipoteses-456611.Spotify.track_in_spotify`
GROUP BY 
  musica_artista
HAVING 
  COUNT(*) > 1;
```

-- Remoção da segunda duplicada por ordem de ID  
```sql
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
```sql
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
```sql
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
```sql
SELECT
  MIN(streams) AS min_streams,
  MAX(streams) AS max_streams,
  AVG(streams) AS media_streams,
  STDDEV(streams) AS desvio_padrao_streams
FROM `projeto-02-hipoteses-456611.Spotify.track_in_spotify_sem_duplicatas`;
```

-- Remoção da linha com dado discrepante 
```sql
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

```sql
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
```sql
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

```sql
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

```sql
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

```sql
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

```sql
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
-- Nesta etapa da análise exploratória, os dados foram agrupados por artista, ano de lançamento e faixa de sucesso. Identificou-se que alguns artistas concentram uma grande quantidade de músicas, o que pode impactar diretamente na soma total de streams. Também foi possível observar um crescimento no número de lançamentos nos anos mais recentes, reforçando a relevância dos dados atuais. Ao agrupar por faixa de sucesso, notou-se que as faixas de médio e alto sucesso concentram a maior quantidade de músicas, sugerindo uma distribuição razoavelmente equilibrada entre os níveis de desempenho.

===============================================================
5.2.2 Ver variáveis categóricas
===============================================================

![grafico01](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/grafico01.png)

-- A visualização da variável categórica "artista" revelou que Taylor Swift lidera o número de músicas na base analisada, com ampla vantagem sobre os demais artistas. Isso indica uma presença forte da artista no catálogo do Spotify em 2023, o que pode influenciar diretamente no volume total de streams. Outros nomes como The Weeknd, Bad Bunny e SZA também aparecem com uma quantidade significativa de faixas. Esse tipo de análise permite identificar quais artistas dominam em termos de produção musical.

===============================================================
5.2.3 Aplicar medidas de tendência central
===============================================================

![tendencia_central](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/tendencia_central.png)

-- Foram aplicadas medidas de tendência central à variável `in_spotify_playlists`, revelando uma média de 5.202 playlists por música. No entanto, a mediana foi significativamente menor (2.209), indicando uma distribuição assimétrica com presença de outliers. Isso sugere que há músicas com presença extremamente alta em playlists, que puxam a média para cima. A análise evidencia que a maioria das faixas tem alcance mais modesto, reforçando a importância de considerar mediana e não apenas a média.

===============================================================
5.2.4 Ver distribuição
===============================================================

-- Codigo Python no Power BI para criar o histograma

```python
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

-- O histograma da variável `streams` revelou uma distribuição altamente assimétrica à direita. A maioria das músicas possui menos de 500 milhões de execuções, enquanto poucas faixas ultrapassam a marca de 1 bilhão de streams. Essa distribuição mostra que o sucesso extremo é concentrado em poucos títulos, enquanto a maioria apresenta desempenho mais modesto. Esse padrão reforça a presença de outliers e justifica a análise complementar com mediana e desvio padrão.

===============================================================
5.2.5 Aplicar medidas de dispersão
===============================================================

![desvio_padrao](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/desvio_padrao.png)

-- Foi calculado o desvio padrão da variável `streams`, resultando em aproximadamente 566,82 milhões. Esse valor elevado confirma a grande variabilidade no número de execuções entre as músicas da base. Isso reforça a presença de outliers e a concentração de alto volume de streams em poucas faixas, o que também foi observado no histograma anterior.

===============================================================
5.2.6 Visualizar o comportamento dos dados ao longo do tempo
===============================================================

![longo_dos_anos](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/longo_dos_anos.png)

-- A análise temporal da soma de streams por ano revelou um crescimento acelerado a partir de 2015, com pico evidente em 2022. Esse comportamento reflete o avanço do consumo de música via plataformas digitais, como o Spotify. Antes da década de 2010, os volumes de execução eram significativamente menores. Essa tendência mostra como o streaming se consolidou como principal meio de distribuição musical nos últimos anos.

===============================================================
5.2.7 Calcular quartis, decis ou percentis
===============================================================

```sql
SELECT
  PERCENTILE_CONT(streams, 0.25) OVER() AS Q1,
  PERCENTILE_CONT(streams, 0.50) OVER() AS Q2,
  PERCENTILE_CONT(streams, 0.75) OVER() AS Q3
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_com_musicas_solo`
LIMIT 1;
```
```sql
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

-- As músicas foram segmentadas em quartis com base na variável `streams`, gerando quatro faixas de sucesso: de baixo a alto. A contagem de registros em cada quartil está equilibrada, com aproximadamente 237 a 238 músicas por faixa. Essa segmentação é útil para análises comparativas, permitindo observar padrões de características musicais e presença em plataformas entre faixas de menor e maior sucesso.

===============================================================
5.2.8 Calcular correlação entre variáveis
===============================================================

```sql
SELECT
  CORR(streams, total_playlists) AS correlacao_streams_playlists
FROM `projeto-02-hipoteses-456611.Spotify.analise_final_quartis`;
```
-- Foi calculada a correlação entre o número de execuções (streams) e o número total de playlists em que a música aparece (total_playlists), utilizando o comando CORR() no BigQuery. O valor encontrado foi 0,63, o que representa uma correlação positiva moderada-forte. Isso indica que quanto mais playlists uma música integra, maior tende a ser sua popularidade (em número de streams), reforçando o papel das playlists como canal estratégico de visibilidade nas plataformas de streaming.

## 5.3 Aplicar técnica de análise

===============================================================
5.3.1 Aplicar segmentação
===============================================================

![segmentacao](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/segmentacao.png)

-- Ao segmentar as músicas por quartis de streams, observamos que faixas classificadas como de alto sucesso (Q4) apresentam, em média, maior BPM e maior nível de energia do que aquelas nos quartis inferiores. Isso indica que músicas mais rápidas e energéticas tendem a se destacar mais em popularidade, reforçando a relevância desses atributos técnicos no desempenho das faixas nas plataformas de streaming.

===============================================================
5.3.2 Validar hipótese
===============================================================

#### 1. Músicas com BPM mais altos fazem mais sucesso em streams no Spotify

![hipotese01](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese01.png)

-- A hipótese de que músicas com BPM mais altos fazem mais sucesso em termos de streams foi refutada. A análise de correlação entre bpm e streams resultou em -0,002, o que indica ausência de relação linear entre as variáveis. O gráfico de dispersão reforça essa conclusão, mostrando uma distribuição aleatória de pontos, sem tendência ascendente ou descendente. Isso sugere que o ritmo da música (em batidas por minuto) não é um fator determinante no sucesso em número de streams no Spotify.

#### 2. As músicas mais populares no ranking do Spotify também possuem um comportamento semelhante em outras plataformas como Deezer.

![hipotese02](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese02.png)

-- A hipótese de que músicas populares no ranking do Spotify também têm bom desempenho no Deezer foi validada. A correlação entre as variáveis in_spotify_charts e in_deezer_charts foi de 0,60, indicando uma relação positiva moderada. O gráfico de dispersão também sugere uma tendência geral: músicas que aparecem frequentemente nas paradas do Spotify também figuram entre as mais tocadas do Deezer, ainda que com variações. Isso reforça que há comportamento semelhante entre as plataformas, mas também que existem particularidades em cada uma.

#### 3. A presença de uma música em um maior número de playlists está relacionada a um maior número de streams.

![hipotese03](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese03.png)

-- A hipótese de que músicas presentes em um maior número de playlists acumulam mais streams foi validada. A correlação entre total_playlists e streams foi de 0,63, o que indica uma relação positiva moderada-forte. O gráfico de dispersão complementa essa análise, mostrando que as faixas inseridas em muitas playlists tendem a atingir altos volumes de reprodução. Isso reforça a importância estratégica das playlists como impulsionadoras do sucesso musical nas plataformas de streaming.

#### 4. Artistas com maior número de músicas no Spotify têm mais streams.

![hipotese04](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese04.png)

-- A hipótese de que artistas com maior número de músicas no Spotify acumulam mais streams foi confirmada. A correlação entre o número de faixas por artista (total_musicas) e o total de execuções (total_streams) foi de 0,78, indicando uma relação forte e positiva. Após corrigir o gráfico de dispersão com o agrupamento por artista, foi possível visualizar que, de fato, os artistas com mais faixas publicadas tendem a alcançar maiores volumes de audiência, sugerindo que a produtividade artística está relacionada ao sucesso nas plataformas de streaming.

#### 5. As características da música influenciam no sucesso em termos de streams no Spotify.

![hipotese05](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/hipotese05.png)

-- A hipótese de que as características técnicas da música influenciam diretamente no sucesso em termos de streams foi refutada com base nos dados analisados. As correlações calculadas entre streams e variáveis como energy, danceability, valence, acousticness, entre outras, apresentaram valores muito baixos e todos negativos, como por exemplo danceability (-0,10) e speechiness (-0,11). Esses resultados indicam que, para a base de dados analisada, não há relação linear relevante entre essas características e o desempenho das faixas em número de execuções.


## 5.4 Resumir informações em um dashboard ou relatório

![relatorio_geral](https://github.com/Mariaamorim25/Projeto-02/blob/main/IMAGENS/relatorio_geral.png)





















