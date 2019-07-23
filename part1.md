- Enviar consultas de Hive diretamente na Linha de Comando do Hadoop
hive -e "<hive query>";

-- Enviar consultas de Hive em arquivos de .hql
hive -f "<path to the .hql file>"

-- Suprimir a impressão da tela de status de progresso de consultas de Hive
hive -S -f "<path to the .hql file>"
hive -S -e "<Hive queries>"

-- Resultados da consulta de Hive em um arquivo local de saída
hive -e "<hive query>" > <local path in the head node>

Criar banco de dados e tabelas Hive
-- Create Hive database and tables
create database if not exists <database name>;
create external table if not exists <database name>.<table name>
(
	field1 string, 
	field2 int, 
	field3 float, 
	field4 double, 
	...,
	fieldN string
) 
row format delimited fields terminated by '<field separator>' 
lines terminated by '<line separator>' 
stored as textfile location '<storage location>'
tblproperties("skip.header.line.count"="1");

<database name> : o nome do banco de dados que você deseja criar. Se quiser apenas usar o banco de dados padrão, a consulta create database... poderá ser omitida.
<table name> : o nome da tabela que você deseja criar no banco de dados especificado. Caso deseje usar o banco de dados padrão, a tabela poderá ser referida diretamente pelo <nome da tabela> sem o <nome do banco de dados>.
<field separator> : o separador que delimita os campos no arquivo de dados a serem carregados na tabela do Hive.
<line separator> : o separador que delimita as linhas no arquivo de dados.
<storage location> : o local de armazenamento do Azure para salvar os dados das tabelas do Hive. Se você não especificar LOCATION <local de armazenamento> , o banco de dados e as tabelas serão armazenados no diretório hive/warehouse/ no contêiner padrão do cluster do Hive por padrão. Se você quiser especificar a localização de armazenamento, esta deverá estar dentro do contêiner padrão para o banco de dados e tabelas. Esse local deve ser referenciado como local relativo ao contêiner padrão do cluster no formato ' WASB:///<Directory 1 >/' ou '<WASB:///Directory 1 >/<Directory 2 >/' , etc. Após a consulta ser executada, os diretórios relativos serão criados no contêiner padrão.
TBLPROPERTIES("skip.header.line.count"="1") : Se o arquivo de dados tiver uma linha de cabeçalho, você precisará adicionar essa propriedade ao final da consulta create table. Caso contrário, a linha de cabeçalho será carregada como um registro para a tabela. Se o arquivo de dados não tiver uma linha de cabeçalho, essa configuração pode ser omitida na consulta.

Carregar dados para tabelas Hive
-- Load data into Hive tables
load data inpath '<path to data>'
into table <database name>.<table name>;

Criar tabela Hive particionada no formato ORC
Se o volume de dado é grande, particionar a tabela é útil para consultas que só 
precisam verificar algumas partições da tabela. Por exemplo, é útil particionar
os dados de log de um site da Web por datas.
Além do particionamento de tabelas Hive, também é útil armazenar os dados Hive
no formato ORC.

The Optimized Row Columnar (ORC) file format provides a highly efficient way to
store Hive data. It was designed to overcome limitations of the other Hive file
formats. Using ORC files improves performance when Hive is reading, writing, and
processing data.

ORC file format has many advantages such as:

a single file as the output of each task, which reduces the NameNode's load
Hive type support including datetime, decimal, and the complex types (struct, list, map, and union)
light-weight indexes stored within the file
skip row groups that don't pass predicate filtering
seek to a given row
block-mode compression based on data type
run-length encoding for integer columns
dictionary encoding for string columns
concurrent reads of the same file using separate RecordReaders
ability to split files without scanning for markers
bound the amount of memory needed for reading or writing
metadata stored using Protocol Buffers, which allows addition and removal of fields

-- Create partitioned Hive tables and load data by partition
create external table if not exists <database name>.<table name>
(
	field1 string,
	...
	fieldn string
)
partitioned by (<partitionfieldname> vartype)
row format delimited fields terminated by '<field separator>'
lines terminated by '<line separator>'
tblproperties("skip.header.line.count"="1");

load data inpath '<path to the source file>'
into table <database name>.<partitioned table name> 
partition (<partitionfieldname>=<partitionfieldvalue>);

HiveQL Syntax
File formats are specified at the table (or partition) level. You can specify
the ORC file format with HiveQL statements such as these:

- CREATE TABLE ... STORED AS ORC
- ALTER TABLE ... [PARTITION partition_spec] SET FILEFORMAT ORC
- SET hive.default.fileformat=Orc

-- Query from Hive tables with partitions
select 
	field1, field2, ..., fieldN
from <database name>.<partitioned table name> 
where <partitionfieldname>=<partitionfieldvalue> and ...;

-- Store Hive data in ORC format
-- First, create a table stored as textfile and load data to the table
create external table if not exists <database name>.<external textfile table name>
(
	field1 string,
	field2 int,
	...
	fieldn date
)
row format delimited fields terminated by '<field separator>' 
lines terminated by '<line separator>' stored as textfile 
location 'wasb:///<directory in azure blob>'
tblproperties("skip.header.line.count"="1");

load data inpath '<path to the source file>'
into table <database name>.<table name>;

-- Second, create a table stored as ORC.
create table if not exists <database name>.<orc table name> 
(
	field1 string,
	field2 int,
	...
	fieldn date
) 
row format delimited fields terminated by '<field separator>'
stored as orc;

-- Third, insert the records from the textfile format table to the
-- ORC format table
insert overwrite table <database name>.<orc table name>
select * from <database name>.<external textfile table name>;

-- Finally, drop the textfile format table to save storage
drop table if exists <database name>.<external textfile table name>;

