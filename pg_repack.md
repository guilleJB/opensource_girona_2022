# Combatre el BLOAT de la BD de PostgreSQL amb PG_REPACK

* Eina opensource per compactar l'espai de les taules i indexos de la BD
* Més efectiu que l'AUTOVACCUM de postgres i controlat ja que s'executa de forma manual



## El problema del BLOAT

Quan en una taula hi ha molts `updates` o `deletes` el que passa a nivell de registres al postgres és que manté els registres "actualitzats" o "esborrats" per tal que si s'ha de fer un `ROLLBACK` de la transacció sigui fàcil tornar enrera. 

En cas que es faci `COMMIT` aquests registres es deixen allà a que passi l'`AUTOVACCUM` i faci neteja. En el cas que que la taula tingui molta concurrencia aquesta acció no acaba de fer prou net i queden registres bruts

Amb el temps, aquest espai es pot acumular i provocar una degradació del rendiment tant de les taules com dels índexs. Aquesta acumulació es coneix com a `bloated` taules o índexs.


### Com trobar el bloat 

A la wiki de PostgreSQL https://wiki.postgresql.org/wiki/Show_database_bloat podem trobar una consulta per veure el bloat

Si no funciona podem provar amb la següent consulta:

```sql
WITH constants AS (
  SELECT current_setting('block_size')::numeric AS bs, 23 AS hdr, 4 AS ma
), bloat_info AS (
  SELECT
    ma,bs,schemaname,tablename,
    (datawidth+(hdr+ma-(case when hdr%ma=0 THEN ma ELSE hdr%ma END)))::numeric AS datahdr,
    (maxfracsum*(nullhdr+ma-(case when nullhdr%ma=0 THEN ma ELSE nullhdr%ma END))) AS nullhdr2
  FROM (
    SELECT
      schemaname, tablename, hdr, ma, bs,
      SUM((1-null_frac)*avg_width) AS datawidth,
      MAX(null_frac) AS maxfracsum,
      hdr+(
        SELECT 1+count(*)/8
        FROM pg_stats s2
        WHERE null_frac<>0 AND s2.schemaname = s.schemaname AND s2.tablename = s.tablename
      ) AS nullhdr
    FROM pg_stats s, constants
    GROUP BY 1,2,3,4,5
  ) AS foo
), table_bloat AS (
  SELECT
    schemaname, tablename, cc.relpages, bs,
    CEIL((cc.reltuples*((datahdr+ma-
      (CASE WHEN datahdr%ma=0 THEN ma ELSE datahdr%ma END))+nullhdr2+4))/(bs-20::float)) AS otta
  FROM bloat_info
  JOIN pg_class cc ON cc.relname = bloat_info.tablename
  JOIN pg_namespace nn ON cc.relnamespace = nn.oid AND nn.nspname = bloat_info.schemaname AND nn.nspname <> 'information_schema'
), index_bloat AS (
  SELECT
    schemaname, tablename, bs,
    COALESCE(c2.relname,'?') AS iname, COALESCE(c2.reltuples,0) AS ituples, COALESCE(c2.relpages,0) AS ipages,
    COALESCE(CEIL((c2.reltuples*(datahdr-12))/(bs-20::float)),0) AS iotta -- very rough approximation, assumes all cols
  FROM bloat_info
  JOIN pg_class cc ON cc.relname = bloat_info.tablename
  JOIN pg_namespace nn ON cc.relnamespace = nn.oid AND nn.nspname = bloat_info.schemaname AND nn.nspname <> 'information_schema'
  JOIN pg_index i ON indrelid = cc.oid
  JOIN pg_class c2 ON c2.oid = i.indexrelid
)
SELECT
  type, schemaname, object_name, bloat, pg_size_pretty(raw_waste) as waste
FROM
(SELECT
  'table' as type,
  schemaname,
  tablename as object_name,
  ROUND(CASE WHEN otta=0 THEN 0.0 ELSE table_bloat.relpages/otta::numeric END,1) AS bloat,
  CASE WHEN relpages < otta THEN '0' ELSE (bs*(table_bloat.relpages-otta)::bigint)::bigint END AS raw_waste
FROM
  table_bloat
    UNION
SELECT
  'index' as type,
  schemaname,
  tablename || '::' || iname as object_name,
  ROUND(CASE WHEN iotta=0 OR ipages=0 THEN 0.0 ELSE ipages/iotta::numeric END,1) AS bloat,
  CASE WHEN ipages < iotta THEN '0' ELSE (bs*(ipages-iotta))::bigint END AS raw_waste
FROM
  index_bloat) bloat_summary
ORDER BY raw_waste DESC, bloat DESC
```

El que ens permet es obtenir un resultat on trobarem per cada taula quin és el seu bloat i l'index del bloat d'aquella taula. També apareixen els indexos afectats.

### I Ara que fem?

Doncs principalment s'hauria de fer una revisió dels registres de la consulta que tinguin un `BLOAT` de més de 10MB i planificar per fer una recompactació.

### Com funciona a grans trets

El procés del `pg_repack` el que fa és crear la taula en segon pla (escrivint les dades en disc) i després canvia el relfilenode (seria com l'inode) i esborra la taula vella, per la qual cosa la taula i els índexs es regeneren

### Passos

* Afegir el repository de `postgresql` al nostre sistema
* Instal·lar l'extensió amb `apt-get install postgresql-XX-repack` on XX es la versió del postgres que tenim a partir de la 9.5 
* Instal·lar la extensió a la BD que volem utilitzar
* Llançar la commanda més típica seria  `pg_repack -d BD  -t TABLENAME -D` tot i que https://reorg.github.io/pg_repack/#usage trobareu molta més informació
* Si es possible fer-ho amb l'accés a la BD aturat sino l'aplicació amb el mínim volum d'accés.

**ALERTA**
Només es pot fer `pg_repack` a taules que tinguin **`PRIMARY KEY`** O **`UNIQUE KEYS`**


## Exemples en productiu després de passar el `pg_repack`

Aquí us indico un parell d'exemples de BD en productiu on es va passar el procés:

   * BD abans del pg_repack 334GB a 138GB --> 196GB nets
   * BD abans del pg_repack 366GB a 160GB --> 206GB nets

## Links d'interes
 - https://reorg.github.io/pg_repack/
 - https://wiki.postgresql.org/wiki/Show_database_bloat
 - https://www.google.com/search?q=pg_repack  XD
