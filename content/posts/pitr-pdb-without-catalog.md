---
title: PDB Point in time recovery (PITR) desde backup
date: 2026-05-03
draft: false
tags:
  - multitenant
  - backup
---

---
El objetivo de esta prueba de concepto es recuperar una pdb a un punto en concreto en el tiempo teniendo en cuenta que no existe catálogo y hay 3 pdbs dentro de una cdb.

Al no tener flashback activado, el flashback database no se puede hacer.

Al ser un entorno productivo no quiero causar afectación en la base de datos origen, por lo que la prueba que he hecho es creando una cdb auxiliar para restaurar desde backup a un punto en concreto en el tiempo.

Esto requiere espacio extra para poder hacer la recuperación de la base de datos. 

## Punto de partida
La versión de rdbms con la que voy a hacer estas pruebas es 19.29.

Se parte de una cdb con tres pdbs. El handicap es que solo quiero restaurar una de las pdb, ya que hacer la cdb entera necesitaría más espacio de almacenamiento y tiempos de restore. 

CDB1
- PDB1
- PDB2
- PDB3

Utilizando la lista anterior a modo de ejemplo quiero restaurar únicamente la PDB3. 
## Creación CDB
El primer paso es crear una CDB auxiliar donde restauraré posteriormente el backup.

Para la creación de la CDB he utilizado los siguientes ficheros:

**Fichero response**
```sh
cat > create-cdb.rsp
sid=cdbaux
gdbName=cdbaux
createAsContainerDatabase=true
numberOfPDBs=0
databaseConfigType=RAC
nodelist=node1,node2
databaseType=MULTIPURPOSE
templateName=/ruta/cdb-create.dbc
datafileJarLocation={ORACLE_HOME}/assistants/dbca/templates/
datafileDestination=+DATA
storageType=ASM
redoLogFileSize=2048
useLocalUndoForPDBs=true
enableArchive=true
recoveryAreaDestination=+FRA
recoveryAreaSize=266240
memoryMgmtType=AUTO
totalMemory=8192
createListener=LISTENER:1521
characterSet=AL32UTF8
nationalCharacterSet=AL16UTF16
sysPassword=******
systemPassword==******
sampleSchema=false
initParams=log_archive_format=%t_%s_%r.arc
```

**Fichero template**
```sh
cat /ruta/cdb-create.dbc
<?xml version = '1.0'?>
<DatabaseTemplate name="General Purpose" description=" " version="19.0.0.0.0">
   <CommonAttributes>
      <option name="OMS" value="true" includeInPDBs="true"/>
      <option name="JSERVER" value="true" includeInPDBs="true"/>
      <option name="SPATIAL" value="false" includeInPDBs="false"/>
      <option name="IMEDIA" value="false" includeInPDBs="false"/>
      <option name="ORACLE_TEXT" value="true" includeInPDBs="true">
         <tablespace id="SYSAUX"/>
      </option>
      <option name="SAMPLE_SCHEMA" value="false" includeInPDBs="false"/>
      <option name="CWMLITE" value="true" includeInPDBs="true">
         <tablespace id="SYSAUX"/>
      </option>
      <option name="APEX" value="false" includeInPDBs="false"/>
      <option name="DV" value="true" includeInPDBs="true"/>
   </CommonAttributes>
   <Variables/>
   <CustomScripts Execute="false"/>
   <InitParamAttributes>
      <InitParams>
         <initParam name="db_name" value=""/>
         <initParam name="dispatchers" value="(PROTOCOL=TCP) (SERVICE={SID}XDB)"/>
         <initParam name="audit_file_dest" value="{ORACLE_BASE}/admin/{DB_UNIQUE_NAME}/adump"/>
         <initParam name="compatible" value="19.0.0"/>
         <initParam name="remote_login_passwordfile" value="EXCLUSIVE"/>
         <initParam name="undo_tablespace" value="UNDOTBS1"/>
         <initParam name="control_files" value="(&quot;{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/control01.ctl&quot;, &quot;{ORACLE_BASE}/fast_recovery_area/{DB_UNIQUE_NAME}/control02.ctl&quot;)"/>
         <initParam name="diagnostic_dest" value="{ORACLE_BASE}"/>
         <initParam name="audit_trail" value="db"/>
         <initParam name="db_block_size" value="8" unit="KB"/>
         <initParam name="open_cursors" value="300"/>
      </InitParams>
      <MiscParams>
         <databaseType>MULTIPURPOSE</databaseType>
         <maxUserConn>20</maxUserConn>
         <percentageMemTOSGA>40</percentageMemTOSGA>
         <customSGA>false</customSGA>
         <dataVaultEnabled>false</dataVaultEnabled>
         <archiveLogMode>false</archiveLogMode>
         <initParamFileName>{ORACLE_BASE}/admin/{DB_UNIQUE_NAME}/pfile/init.ora</initParamFileName>
      </MiscParams>
      <SPfile useSPFile="true">{ORACLE_HOME}/dbs/spfile{SID}.ora</SPfile>
   </InitParamAttributes>
   <StorageAttributes>
      <DataFiles>
         <Location>{ORACLE_HOME}/assistants/dbca/templates/Seed_Database.dfb</Location>
         <SourceDBName cdb="true">seeddata</SourceDBName>
         <Name id="3" Tablespace="SYSAUX" Contents="PERMANENT" Size="512" autoextend="true" blocksize="8192" con_id="1">{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/sysaux01.dbf</Name>
         <Name id="1" Tablespace="SYSTEM" Contents="PERMANENT" Size="2048" autoextend="true" blocksize="8192" con_id="1">{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/system01.dbf</Name>
         <Name id="4" Tablespace="UNDOTBS1" Contents="UNDO" Size="2048" autoextend="true" blocksize="8192" con_id="1">{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/undotbs01.dbf</Name>
         <Name id="7" Tablespace="USERS" Contents="PERMANENT" Size="512" autoextend="true" blocksize="8192" con_id="1">{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/users01.dbf</Name>
      </DataFiles>
      <TempFiles>
         <Name id="1" Tablespace="TEMP" Contents="TEMPORARY" Size="2048" con_id="1">{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/temp01.dbf</Name>
      </TempFiles>
      <ControlfileAttributes id="Controlfile">
         <maxDatafiles>100</maxDatafiles>
         <maxLogfiles>16</maxLogfiles>
         <maxLogMembers>3</maxLogMembers>
         <maxLogHistory>1</maxLogHistory>
         <maxInstances>8</maxInstances>
         <image name="control01.ctl" filepath="{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/"/>
         <image name="control02.ctl" filepath="{ORACLE_BASE}/fast_recovery_area/{DB_UNIQUE_NAME}/"/>
      </ControlfileAttributes>
      <RedoLogGroupAttributes id="1">
         <reuse>false</reuse>
         <fileSize unit="KB">204800</fileSize>
         <Thread>1</Thread>
         <member ordinal="0" memberName="redo01.log" filepath="{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/"/>
      </RedoLogGroupAttributes>
      <RedoLogGroupAttributes id="2">
         <reuse>false</reuse>
         <fileSize unit="KB">204800</fileSize>
         <Thread>1</Thread>
         <member ordinal="0" memberName="redo02.log" filepath="{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/"/>
      </RedoLogGroupAttributes>
      <RedoLogGroupAttributes id="3">
         <reuse>false</reuse>
         <fileSize unit="KB">204800</fileSize>
         <Thread>1</Thread>
         <member ordinal="0" memberName="redo03.log" filepath="{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/"/>
      </RedoLogGroupAttributes>
   </StorageAttributes>
</DatabaseTemplate>
```

Para crear la CDB utilizo dbca con el response que he puesto anteriormente con el siguiente comando:
```sh
nohup $ORACLE_HOME/bin/dbca -J-Doracle.assistants.dbca.validate.ConfigurationParams=false -silent -createDatabase -responseFile /ruta/create-cdb.rsp &
```

El proceso de creación de la CDB ha tardardo 1h 55min

Los requisitos para esta operativa es tener local undo enabled en la pdb y modo archivelog.

## Prueba de restore
Al estar el backup de la base de datos original almacenado en cinta, se deben poner los parámetros de cinta en la conexión auxiliar.

>Nota: tener en cuenta poner la base de datos en modo exclusivo antes de lanzar el restore.

```sql
alter system set cluster_database=false scope=spfile;
```

>Nota2: añadir en la entrada de tnsnames de la base de datos auxiliar (UR=A)

Cargar las variables de la cdb auxiliar que se ha creado anteriormente.

Ahora llega el momento de lanzar el duplicate con la cadena de conexión siguiente, se accede como target a la base de datos origen cdb1 y como auxiliary a la cdbaux.

Esto es necesario debido a que no se tiene catálogo para hacer el restore directamente. Conectamos a la cdb original para utilizarla como puente y conocer la ubicación de los backups.

```sql
rman target sys/password@CDB1 auxiliary /

RUN {
ALLOCATE CHANNEL c1 TYPE SBT_TAPE maxopenfiles 4 parms 'ENV=(********)';
ALLOCATE AUXILIARY CHANNEL aux1 TYPE SBT_TAPE maxopenfiles 4 parms 'ENV=(********)';
SET UNTIL TIME "TO_DATE('2026-04-13 12:00:00','YYYY-MM-DD HH24:MI:SS')";
DUPLICATE DATABASE CDB1 TO CDBAUX PLUGGABLE DATABASE PDB3 NOFILENAMECHECK NOOPEN;
}
```

Una vez acabado el duplicate, abro la pdb y accedo a la que acabo de restaurar:
```sql
sqlplus / as sysdba

ALTER DATABASE OPEN RESETLOGS;
ALTER PLUGGABLE DATABASE PDB3 OPEN;
ALTER SESSION SET CONTAINER = PDB3;
```

En este punto ya está la pdb restaurada a un punto concreto en el tiempo. Ahora se puede revisar y ver lo necesario.

## Conclusión

Con esta prueba de concepto he conseguido restaurar una única pdb de una cdb con tres pdbs en una cdb auxiliar. Sin causar afectación en producción y con la base de datos en un punto en concreto del tiempo donde puedo ver los datos en la fecha propuesta.

---

*Las opiniones y contenidos de este blog son míos y no representan a Oracle.*


