# Creación de aplicación web en contenedores con Docker

Bienvenidos a la guía para la proteccion de VM's mediante Azure Backup, este resumen practico esta hecho gracias a la documentación oficial de Microsoft https://learn.microsoft.com/en-us/training/modules/protect-virtual-machines-with-azure-backup/?wt.mc_id=certnurture_StayCurrent30DayReminder_email_wwl&ns-enrollment-type=Collection Realizada para practicar para la renovación de la certificación **AZ-104** a **Marzo de 2024**.




<img align="center" src="/img/1ºimagenn.PNG"  />

## Conceptos básicos

### Azure Backup

Servicio integrado en Azure que proporciona una seguridad confiable para todos los activos de datos gestionados por la plataforma. Ofrece soluciones sin infraestructura para facilitar copias de seguridad y restauraciones automáticas, con administración a escala y costos predecibles. Actualmente, Azure Backup ofrece soluciones especializadas para copias de seguridad de VM's tanto en Azure como localmente. También proporciona opciones de copia de seguridad y restauración de nivel empresarial para cargas de trabajo críticas como SQL Server o SAP HANA en máquinas virtuales de Azure

### Azure Backup vs Site Recovery

Azure Backup se diferencia de Azure Site Recovery en su enfoque: mientras que Backup se centra en mantener copias de datos para retroceder en el tiempo, Site Recovery replica datos casi en tiempo real y facilita la conmutación por error en caso de fallos. Site Recovery se utiliza para desastres a gran escala, mientras que las copias de seguridad son útiles para la pérdida accidental de datos, corrupción o ataques de ransomware.

La elección entre Azure Backup y Azure Site Recovery depende de la criticidad de la aplicación, los objetivos de punto de recuperación (RPO) y tiempo de recuperación (RTO), y las consideraciones de costos

### Ventajas

Azure Backup ofrece múltiples ventajas sobre las soluciones tradicionales, como eliminación de la infraestructura de copia de seguridad, retención a largo plazo de datos, seguridad con control de acceso basado en roles y cifrado de datos, así como alta disponibilidad con opciones de replicación y monitoreo centralizado.

Soporta varios escenarios, incluyendo copias de seguridad de máquinas virtuales en Azure, copias de seguridad locales mediante el uso de agentes o servidores dedicados, copias de seguridad de recursos compartidos de Azure Files y soluciones especializadas para copias de seguridad de SQL Server y SAP HANA en máquinas virtuales de Azure.

### Recovery Services Vault

Esta bóveda funciona como un sistema centralizado de gestión de almacenamiento, simplificando las operaciones de copia de seguridad y restauración. Con Azure Backup, no es necesario preocuparse por configurar o administrar cuentas de almacenamiento; simplemente se debe especificar la bóveda donde se realizarán las copias de seguridad de las máquinas virtuales. Los datos de respaldo se transfieren automáticamente a las cuentas de almacenamiento de Azure Backup en un dominio de error independiente en segundo plano. Además de su función de almacenamiento, la bóveda actúa como un límite de control de acceso basado en roles (RBAC), asegurando un acceso seguro a los datos de copia de seguridad

### Instantáneas

Una instantánea es una copia de seguridad de un momento dado de todos los discos de la máquina virtual. Para las máquinas virtuales de Azure, Azure Backup usa diferentes extensiones para cada sistema operativo compatible:

Ampliar tabla

| Extensión       | SO      | Descripción                                                  |
| :-------------- | :------ | :----------------------------------------------------------- |
| VMSnapshot      | Windows | La extensión funciona con el Servicio de instantáneas de volumen (VSS) para realizar una copia de los datos en el disco y en la memoria. |
| VMSnapshotLinux | linux   | La instantánea es una copia del disco.                       |

1. **Consistente con la aplicación:**
   - El snapshot captura la VM en su totalidad. Utiliza escritores de VSS para capturar el contenido de la memoria de la máquina y cualquier operación de E/S pendiente.
   - Para máquinas Linux, es necesario escribir scripts personalizados previos o posteriores por aplicación para capturar el estado de la aplicación.
   - Se puede obtener una consistencia completa para la VM y todas las aplicaciones en ejecución.
2. **Consistente con el sistema de archivos:**
   - Si VSS(Volume Shadow Copy Service) falla en Windows o si los scripts previos y posteriores fallan en Linux, Azure Backup aún crea un snapshot consistente con el sistema de archivos.
   - Durante la recuperación, no hay corrupción dentro de la máquina. Sin embargo, las aplicaciones instaladas pueden necesitar realizar su propia limpieza durante el inicio para lograr consistencia.
3. **Consistente en caso de fallo (crash consistent):**
   - Este nivel de consistencia ocurre típicamente si la VM se apaga durante el proceso de respaldo.
   - No se capturan operaciones de E/S ni contenidos de memoria durante este método de respaldo.
   - Este método no garantiza la consistencia de datos para el sistema operativo o las aplicaciones.

### Politicas de Respaldo o Backup Policies

Puedes definir la frecuencia de respaldo y la duración de retención para tus copias de seguridad. Actualmente, el respaldo de VM se puede activar diariamente o semanalmente, y se puede almacenar durante varios años. La política de respaldo admite dos niveles de acceso: nivel de instantánea y nivel de bóveda.

- Nivel de instantánea: Todas las instantáneas se almacenan localmente por un período máximo de cinco días. Esto se conoce como el nivel de instantánea. Para todas las recuperaciones de operaciones, recomendamos que restaures desde las instantáneas porque es más rápido hacerlo así. Esta capacidad se llama restauración instantánea.
- Nivel de bóveda: Todas las instantáneas también se transfieren a la bóveda para obtener más seguridad y una retención más larga. En este punto, el tipo de punto de recuperación cambia a "instantánea y bóveda".

### Ejercicio: Creación de copia de seguridad para VM's

#### Para realizar este ejercicio necesitamos una suscripcion real.

Crearemos un grupo de recursos que contenga los recursos necesarios, abriremos nuestro cloud shell 

```
RGROUP=$(az group create: --name vmbackups --location westus2 --output tsv --query name)
```

Crearemos una red virtual llamada `NorthwindInternal` y una subred llamada `NorthwindInternal1`:

```
az network vnet create \
--resource-group $RGROUP \
--name NorthwindInternal \
--address-prefixes 10.0.0.0/16 \
--subnet-name NorthwindInternal1 \
--subnet-prefixes 10.0.0.0/24
```

A continuación crearemos una maquina virtual de Windows llamada `NW-APP01`, usaremos una imagen de win2016Datacenter, asociaremos la red y subred creada anteriormente, el siguente sky y size:

```
az vm create \
--resource-group $RGROUP \
--name NW-APP01 \
--size Standard_DS1_v2 \
--public-ip-sku Standard \
--vnet-name NorthwindInternal \
--subnet NorthwindInternal1 \
--image Win2016Datacenter \
--admin-username admin123 \
--no-wait \
--admin-password PassWord123!
```

Ahora crearemos la maquina virtual de Linux, se llamara `NW-RHEL01` y dispondrá de las siguientes características:

```
az vm create \
--resource-group $RGROUP \
--name NW-RHEL01 \
--size Standard_DS1_v2 \
--image RedHat:RHEL:7-RAW:latest \
--authentication-type ssh \
--generate-ssh-keys \
--vnet-name NorthwindInternal \
--subnet NorthwindInternal1
```

#### Habilitar copia de seguridad para las VM's

Vamos al menu de nuestra maquina virtual desde recursos o buscando maquinas virtuales en el portal y seleccionamos lo siguiente, para crear la vault o boveda de manera standard:

<img align="center" src="/img/2ºimagenn.PNG"  />

Una vez que se complete la implementación, regrese a la máquina virtual **NW-RHEL01** , desplácese hacia abajo hasta **Copia de seguridad + recuperación ante desastres** y seleccione **Copia de seguridad** . Aparece el panel **Copia de seguridad** para la máquina virtual *NW-RHEL01* y crearemos la primera copia

Ahora a través del cloudshell para la otra maquina, creamos la bóveda o vault:

```
az backup vault create \
--resource-group vmbackups \
--name azure-backup
```

Habilitamos la copia de seguridad, asociando el vault y la maquina virtual, podemos ver que ahora hemos seleccionado Enhanced en vez de Standard :

```
az backup protection enable-for-vm \
--resource-group vmbackups \
--vault-name azure-backup º
--vm NW-APP01 \
--policy-name EnhancedPolicy
```

Realizamos la copia de seguridad inicial

```
az backup protection backup-now \
--resource-group vmbackups \
--vault-name azure-backup \
--container-name NW-APP01 \
--item-name NW-APP01 \
--retain-until 18-10-2030 \
--backup-management-type AzureIaasVM
```

Ya podemos comprobar desde el portal las copias de seguridad.

<img align="center" src="/img/3ºimagenn.PNG"  />
