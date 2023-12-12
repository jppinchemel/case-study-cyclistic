![Cyclistic Bike-Share Logo](https://i.imgur.com/EUXY39x.png)
# Estudo de caso: Cyclistic Bike-Share
O presente estudo de caso é o projeto final do Certificado Profissional de Análise de Dados do Google, com o objetivo de avaliar as habilidades analíticas e ferramentas ensinadas durante a formação. Para isso, foi selecionado o caso acerca da empresa fictícia de compartilhamento de bicicletas “Cyclistic Bike-Share” seguindo o framework de: ask, prepare, process, analyze e act.

## Ask
Nesse cenário, o diretor de marketing acredita que o sucesso da empresa está associado à maximização de clientes que assinem o plano de filiação anual. Além destes, existem também os “clientes casuais” que optam pelos planos diários ou de corrida única. A partir disso, o objetivo desse estudo, como analista de dados, é identificar como os clientes casuais e anuais utilizam o serviço de bicicletas Cyclistic diferentemente. Assim, futuramente converter os clientes casuais em clientes anuais por meio de uma estratégia de marketing.

## Prepare
### Fonte de dados
Para trabalhar com os dados de viagens da empresa, foi utilizado o histórico do primeiro semestre de 2023 de viagens da empresa, disponibilizado no endereço https://divvy-tripdata.s3.amazonaws.com/index.html. Foram utilizadas informações públicas para explorar o comportamento dos clientes, mantendo em privado os dados pessoais dos usuários.

Não há parcialidade no conjunto quando se trata dos próprios clientes da empresa e informações gerais das viagens. Uma vez que os dados são disponibilizados diretamente pela empresa responsável pelo serviço, podemos considerá-los confiáveis, originais, abrangentes, atuais e citados (ROCCC). Além disso, a seguridade dos dados é mantida, preservando informações confidenciais dos usuários.

### Organização dos dados
Os dados estão organizados em 6 diferentes tabelas, uma para cada mês do histórico utilizado, carregadas na plataforma BigQuery do Google Cloud. Nas colunas existem informações sobre o ID da corrida, tipo de bicicleta utilizada, ponto de início e final da corrida, nome e ID da estação de início e estação final, latitude e longitude inicial e final da corrida e se o cliente é ou não casual. Todos os arquivos estão consistentes entre si, em suas colunas e presentam o tipo correto de dado para utilizar em análises (string, float e timestamp). Assim, a partir da análise dos dados oferecidos, é possível identificar os padrões de consumo de clientes casuais e membros anuais, contribuindo para o futuro desenvolvimento de uma estratégia de marketing para a conversão de novos clientes.

![Esquema da tabela no BigQuery](https://i.imgur.com/tGIZyBt.png)

# Process 
### Combinando os documentos
Inicialmente, os 6 documentos .csv foram combinador em apenas uma tabela na plataforma BigQuery:
```
-- União das tabelas
CREATE TABLE `myprojects-cases.tripdata.combined_data` AS 
SELECT *

FROM (
SELECT * FROM `myprojects-cases.tripdata.202301`
UNION ALL
SELECT * FROM `myprojects-cases.tripdata.202302`
UNION ALL
SELECT * FROM `myprojects-cases.tripdata.202303`
UNION ALL
SELECT * FROM `myprojects-cases.tripdata.202304`
UNION ALL
SELECT * FROM `myprojects-cases.tripdata.202305`
UNION ALL
SELECT * FROM `myprojects-cases.tripdata.202306`
);
```

### Familiarização dos dados 
Para compreender como estão dispostos os dados da nova tabela criada, foi analisada a quantidade total de linhas nessa tabela e explorada as informações de suas colunas.

```
-- Consultando o número total de linhas na tabela (2390459)
SELECT COUNT(*) AS num_rows
FROM `myprojects-cases.tripdata.combined_data`

-- Analisando a coluna ride_id
SELECT length(ride_id) 
FROM `myprojects-cases.tripdata.combined_data`

-- Organização dos valores em ride_id
SELECT length(ride_id) 
FROM `myprojects-cases.tripdata.combined_data`
WHERE length(ride_id) != 16

-- Valores duplicados dentro da chave primária
SELECT COUNT(ride_id) - COUNT(DISTINCT ride_id) AS duplicate_rows
FROM `myprojects-cases.tripdata.combined_data`

-- Avaliando a presença de NULL values nas colunas
SELECT 
  COUNT(*) - COUNT(ride_id) AS ride_id_count,
  COUNT(*) - COUNT(rideable_type) AS rideable_type_count,
  COUNT(*) - COUNT(started_at) AS started_at_count,
  COUNT(*) - COUNT(ended_at) AS ended_at_count,
  COUNT(*) - COUNT(start_lat) AS start_lat_count,
  COUNT(*) - COUNT(start_lng) AS start_lng_count,
  COUNT(*) - COUNT(end_lat) AS end_lat_count, -- 2460 null values
	COUNT(*) - COUNT(end_lng) AS end_lng_count, -- 2460 null values
  COUNT(*) - COUNT(start_station_name) AS start_station_name_count, -- 357417 null values
  COUNT(*) - COUNT(start_station_id) AS start_station_id_count, -- 357549 null values
  COUNT(*) - COUNT(end_station_name) AS end_station_name_count, -- 380963 null values
  COUNT(*) - COUNT(end_station_id) AS end_station_id_count, -- 381104 null values
  COUNT(*) - COUNT(member_casual) AS member_casual_count,

FROM `myprojects-cases.tripdata.combined_data`

-- Valores possíveis na coluna rideable_type
SELECT DISTINCT rideable_type, COUNT(*) AS num_rideable_type
FROM `myprojects-cases.tripdata.combined_data`
GROUP BY rideable_type

-- Valores possíveis na coluna member_casual
SELECT DISTINCT member_casual, COUNT(member_casual) AS num_member_type
FROM `myprojects-cases.tripdata.combined_data`
GROUP BY member_casual
```

Inicialmente, foi consultado o número total de linhas presente na tabela, identificando 2390459 linhas. Após isso, a chave primária ride_id foi analisada, verificando o tamanho dessa chave, se há entradas formatadas diferentemente e valores duplicados.

Todos os valores para ride_id possuem 16 caracteres e nenhuma entrada duplicada foi encontrada. Para verificar a presença de null values em cada coluna, foi subtraída a contagem de valores non-null da contagem total de linhas. Além disso, também analisei os dados presentes nas colunas rideable_type e member_casual.

### Manipulação e limpeza
Uma vez entendido como estão organizados os dados em suas colunas e identificado os valores nulls presentes, foi criada uma consulta para adição de novas colunas para posterior análise. Nessa consulta, foram adicionadas as colunas “day_of_week”, “month”, “starting_hour” e “ride_length”. Além disso, foram filtrados também todas as linhas que possuem valores null identificados anteriormente e corridas com tempo inferior a um minuto e superior a um dia.

```

-- Criação de uma nova tabela com novas colunas e valores NULL removidos

CREATE TABLE `myprojects-cases.tripdata.cleaned_combined_data3` AS (
  SELECT
  *,
  CASE
    WHEN EXTRACT(DAYOFWEEK from started_at) = 1 THEN 'Sun'
    WHEN EXTRACT(DAYOFWEEK from started_at) = 2 THEN 'Mon'
    WHEN EXTRACT(DAYOFWEEK from started_at) = 3 THEN 'Tue'
    WHEN EXTRACT(DAYOFWEEK from started_at) = 4 THEN 'Wed'
    WHEN EXTRACT(DAYOFWEEK from started_at) = 5 THEN 'Thu'
    WHEN EXTRACT(DAYOFWEEK from started_at) = 6 THEN 'Fri'
  ELSE 'Sat'
  END AS day_of_week,
  CASE
    WHEN EXTRACT (MONTH FROM started_at) = 1 THEN 'Jan'
    WHEN EXTRACT (MONTH FROM started_at) = 2 THEN 'Feb'
    WHEN EXTRACT (MONTH FROM started_at) = 3 THEN 'Mar'
    WHEN EXTRACT (MONTH FROM started_at) = 4 THEN 'Apr'
    WHEN EXTRACT (MONTH FROM started_at) = 4 THEN 'May'
  ELSE 'Jun'
  END AS month,
  EXTRACT(HOUR from started_at) AS starting_hour,
  TIMESTAMP_DIFF(ended_at, started_at, minute) AS ride_length
  FROM
  `myprojects-cases.tripdata.combined_data`
  WHERE 
  start_station_name IS NOT NULL AND
  start_station_id IS NOT NULL AND
  end_station_name IS NOT NULL AND
  end_station_id IS NOT NULL AND
  end_lng IS NOT NULL AND
  end_lng IS NOT NULL AND
  TIMESTAMP_DIFF(ended_at, started_at, minute) > 1 AND
  TIMESTAMP_DIFF(ended_at, started_at, minute) < 1440
)
```

Ao final do processo, foram recuperadas 1743446 linhas. Dessa forma, 647.013 linhas foram excluídas durante o processo de limpeza dos dados.

```
SELECT COUNT(*)
FROM `myprojects-cases.tripdata.cleaned_combined_data`
```

# Analyze and Share
![Dashboard](https://i.imgur.com/bghXn1l.png)

Após a preparação dos dados pelas etapas anteriores, finalmente pode-se realizar uma análise mais profunda para identificar padrões e relações entre esses dados. Essa análise foi feita por meio do BigQuery e também pode ser visualizada em uma [dashboard criada no Tableau](https://public.tableau.com/app/profile/pinchemel/viz/CaseStudy_17010360684960/CyclisticDashboard?publish=yes).


```
-- Número de corridas para cada tipo de cliente
SELECT member_casual, COUNT(*) as total_members
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY member_casual

-- Quantidade de corridas em cada tipo de bicicleta entre os tipos de clientes
SELECT member_casual, rideable_type, COUNT(*) AS total_trips
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY member_casual, rideable_type
ORDER BY member_casual, total_trips

-- Duração média das corridas entre cada tipo de cliente
SELECT member_casual, AVG(ride_length) AS avg_length
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY member_casual

-- Total de corridas para cada dia da semana de acordo com cada tipo de cliente
SELECT member_casual, day_of_week,COUNT(day_of_week) AS total_rides
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY member_casual, day_of_week
ORDER BY member_casual, day_of_week

-- Início de corridas entre cada tipo de cliente 
SELECT start_station_name, member_casual,
  AVG(start_lat) AS start_lat, AVG(start_lng) AS start_lng, COUNT(start_station_name) AS total_trips
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY start_station_name, member_casual
ORDER BY total_trips DESC

-- Final de corridas entre cada tipo de cliente 
SELECT end_station_name, member_casual,
  AVG(end_lat) AS end_lat, AVG(end_lng) AS end_lng, COUNT(end_station_name) AS total_trips
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY end_station_name, member_casual
ORDER BY total_trips DESC

-- Horário de pico entre os diferentes clientes
SELECT member_casual, starting_hour, COUNT(*) as total_rides
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY member_casual, starting_hour
ORDER BY total_rides DESC;

-- Corridas por mês entre cada tipo de cliente
SELECT member_casual, month, COUNT(*) as total_rides
FROM `myprojects-cases.tripdata.cleaned_combined_data`
GROUP BY member_casual, month;
```

Após a análise dos pontos destacados na pesquisa, é possível apontar alguns dos principais pontos dos perfis de cada tipo de cliente:

| Casuais  | Membros |
| ------------- | ------------- |
| Representam pouco mais de 34% das corridas semestrais.  | Representam 65,7% das corridas semestrais.  |
| Dividem sua preferência de bicicletas entre clássicas (49,9%) e elétricas (43%), enquanto uma minoria utiliza bicicletas ancoradas.  | Preferem bicicletas clássicas, com 61,5%, enquanto elétricas representam 38,5%.  |
| Apresentam uma duração média de corridas de 22 minutos e seus principais destinos são próximos ao cais de Chicago, porto, parque, praias e museus.   | Apresentam duração média de corridas de aproximadamente 12 minutos e seus principais destinos são universidades e centros comerciais.  |
| As corridas tendem a crescer próximo ao verão, duplicando no mês de abril e triplicando em junho.  | As corridas começam a crescer significativamente em abril, alcançando grande pico em junho.  |
| Utilizam mais frequentemente durante os sábados.  | Utilizam mais frequentemente nos dias de semana.  |
| Seu pico de uso ocorre às 17h.  | Apresenta um pico inicial às 8h e um segundo pico às 17h, indicando seu uso, principalmente, no início e final do horário comercial.  |

# Act 
Baseados no insights adquiridos pela análise anterior e a compreensão de como se difere o comportamentos dos clientes da Cyclistic Bike-Share algumas iniciativas podem ser tomadas para atingir o propósito do programa de marketing:

1. Desenvolver campanhas de marketing durante a temporada de verão, destacando os destinos turísticos e atividades de lazer aos finais de semana;
2. Criação de pacotes de aluguel de bicicletas mais longos, atraindo clientes que buscam viagens mais longas;
3. Reforçar estações próximas a pontos turísticos e área de lazer, atingindo diretamente o público alvo casual.

#### O distintivo de conclusão do curso profissional de Data Analytics do Google pode ser conferido no [Credly](https://www.credly.com/badges/54c302ee-5bfb-41cb-a960-5aad90987c75/public_url).
