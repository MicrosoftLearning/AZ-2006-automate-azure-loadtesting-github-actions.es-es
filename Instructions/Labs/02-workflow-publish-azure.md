---
lab:
  title: 'Laboratorio 02: Uso de Acciones de GitHub para Azure para publicar una aplicación web en Azure App Service'
  module: 'Module 2: Implement GitHub Actions for Azure'
---

# Información general

En este laboratorio, aprenderás a implementar un flujo de trabajo de Acciones de GitHub que implemente una aplicación web en Azure App Service.

Después de completar este laboratorio, podrá:

* Implemente un flujo de trabajo de Acciones de GitHub para CI/CD.
* Explicar las características básicas de los flujos de trabajo de Acciones de GitHub

**Tiempo estimado de finalización: 40 minutos**

## Requisitos previos

* Una **cuenta de Azure** con una suscripción activa. Si aún no tiene una, puede solicitar una prueba gratuita en [https://azure.com/free](https://azure.com/free).
    * Un [explorador](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices) compatible con el portal web de Azure.
    * Una cuenta Microsoft o una cuenta de Microsoft Entra con el rol Colaborador o Propietario en la suscripción de Azure. Para más información, consulte [Enumeración de asignaciones de roles de Azure mediante Azure Portal](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal) y [Ver y asignar roles de administrador en Azure Active Directory](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal).
* Una cuenta de GitHub. Si no tiene una cuenta de GitHub que pueda usar para este laboratorio, siga las instrucciones disponibles en [Registrar una nueva cuenta GitHub](https://github.com/join) para crear una.

## Instrucciones

## Ejercicio 1: Importar eShopOnWeb al repositorio de GitHub

En este ejercicio, importarás el repositorio [eShopOnWeb](https://github.com/MicrosoftLearning/eShopOnWeb) a tu cuenta de GitHub. El repositorio se organiza de la siguiente manera:

| Folder | Contenido |
| -- | -- |
| **.ado** | Canalizaciones YAML de Azure DevOps |
| **.devcontainer** | Configuración para desarrollar mediante contenedores (ya sea localmente en VS Code o en GitHub Codespaces) |
| **infraestructura** | Infraestructura de Bicep y ARM como plantillas de código usadas en algunos escenarios de laboratorio |
| **.github** | Definiciones de flujo de trabajo de GitHub de YAML |
| **src** | El sitio web .NET 8 que se usa en los escenarios de laboratorio |

### Tarea 1: Importar el repositorio de eShopOnWeb

1. En tu explorador web ve a GitHub [http://github.com](http://github.com) e inicia sesión con tu cuenta.
1. Inicia el proceso de importación [https://github.com/new/import](https://github.com/new/import).
1. Escribe la siguiente información en la página **Importa el proyecto a GitHub**.

    | Configuración | Acción |
    |--|--|
    | **Dirección URL del repositorio de origen** | Escriba `https://github.com/MicrosoftLearning/eShopOnWeb`. |
    | **Propietario** | Selecciona tu alias de GitHub |
    | **Nombre del repositorio** | Escribe **eShopOnWeb** |
    | **Privacidad** | Después de seleccionar el **Propietario** aparecerán las opciones de privacidad. Seleccione **Público**. |

1. Selecciona **Comenzar importación** y espera a que se complete el proceso de importación.
1. En la página del repositorio selecciona **Configuración**, después selecciona **Acciones > General** en el panel de navegación izquierdo.
1. En la sección **Permisos de acciones** de la página, selecciona la opción **Permitir todas las acciones y flujos de trabajo reutilizables** y después selecciona **Guardar**.

> **NOTA:** El eShopOnWeb es un repositorio grande y puede tardar entre 5 y 10 minutos en finalizar la importación.

## Ejercicio 2: Creación de recursos de Azure y configuración de GitHub 

En este ejercicio, crearás una entidad de seguridad de servicio de Azure para autorizar a GitHub a acceder a tu suscripción a Azure desde las Acciones de GitHub. También revisarás y modificarás el flujo de trabajo de GitHub que crea, prueba e implementa tu sitio web en Azure.

### Tarea 1: crear una entidad de servicio de Azure y guardarla como secreto de GitHub

En esta tarea, crearás un grupo de recursos y una entidad de seguridad de servicio de Azure. GitHub usa la entidad de servicio para implementar la aplicación eShopOnWeb deseada.

1. En tu explorador ve a Azure Portal [https://portal.azure.com](https://portal.azure.com).
1. Abre el **Cloud shell** y selecciona el modo **Bash**. **Nota:** Debes configurar el almacenamiento persistente si es la primera vez que inicias Cloud Shell.
1. Crea un grupo de recursos con el siguiente comando de la CLI `az group create`. Reemplace `<location>` por una región cercana.

    ```
    az group create -n az2006-rg -l <location>
    ```

1. Ejecuta el siguiente comando para registrar el proveedor de recursos para el **Azure App Service** que implementarás más adelante en el laboratorio.

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. Ejecuta el siguiente comando para generar un nombre aleatorio para la aplicación web que implementes en Azure App Service. Copia y guarda el nombre que genera el comando para usarlo más adelante en ese laboratorio.

    ```
    myAppName=az2006app$RANDOM
    echo $myAppName
    ```

1. Ejecuta los siguientes comandos para recuperar tu Id. de suscripción. Asegúrate de copiar y guardar la salida de los comandos, el valor de Id. de suscripción se usa más adelante en este laboratorio.

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

1. Crea una entidad de seguridad de servicio con los siguientes comandos. El primer comando almacena el Id. del grupo de recursos en una variable.

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)

    az ad sp create-for-rbac --name GH-Action-eshoponweb --role contributor --scopes $rgId --json-auth true
    ```

    >**IMPORTANTE:** Este comando da salida a un objeto JSON que contiene los identificadores usados para autenticar en Azure en el nombre de una identidad Microsoft Entra (entidad de servicio). Copia el objeto JSON para usarlo en los siguientes pasos. 

1. En una ventana del explorador, ve a tu repositorio **eShopOnWeb** de GitHub.
1. En la página del repositorio selecciona **Configuración**, luego selecciona **Secretos y variables > Acciones** en el panel de navegación izquierdo.
1. Selecciona **Nuevo secreto de repositorio** y escribe la siguiente información:
    * **NOMBRE**: `AZURE_CREDENTIALS`
    * **Secreto**: escribe el objeto JSON generado al crear la entidad de servicio.
1. Seleccione **Add secret** (Agregar secreto).

### Tarea 2: modificar y ejecutar el flujo de trabajo de GitHub

En esta tarea, modificarás el flujo de trabajo de GitHub *eshoponweb-cicd.yml* proporcionado y lo ejecutarás para implementar la solución en tu propia suscripción.

1. En una ventana del explorador, vuelve al repositorio de GitHub **eShopOnWeb** .
1. Selecciona **<> Código** y, en la rama principal, selecciona **eshoponweb-cicd.yml** en la carpeta **eShopOnWeb/.github/workflows**. Este flujo de trabajo define el proceso de CI/CD para la aplicación eShopOnWeb.

    ![Captura de pantalla que muestra la ubicación del archivo en la estructura de carpetas.](./media/eshop-cid-workflow.png)
1. Selecciona **Editar este archivo**.
1. Cambia los campos de la sección `env:` del archivo por los siguientes valores.

    | Campo | Action |
    |--|--|
    | RESOURCE-GROUP: | `az2006-rg` |
    | LOCATION: | `eastus` (O bien, la región que has seleccionado al crear el grupo de recursos). |
    | TEMPLATE-FILE: | Sin cambios |
    | SUBSCRIPTION-ID: | Tu Id. de suscripción. |
    | WEBAPP-NAME: | El nombre de la aplicación web generado aleatoriamente que has creado anteriormente en el laboratorio. |

1. Lee atentamente el flujo de trabajo, los comentarios se proporcionan para ayudarte a entender los pasos del flujo de trabajo.
1. Quita la marca de comentario de la sección **on** en la parte superior del archivo mediante la eliminación de `#`. El flujo de trabajo se desencadena con cada inserción en la rama principal y también ofrece un desencadenamiento manual (`workflow_dispatch`).
1. Selecciona **Confirmar cambios...** en la parte superior derecha de la página.
1. Aparecerá una ventana emergente. Acepta los valores por defecto (confirmar directamente en la rama principal) y selecciona **Confirmar cambios**. El flujo de trabajo se ejecutará automáticamente.

### Tarea 3: Revisar la ejecución del flujo de trabajo de GitHub

En esta tarea, revisarás la ejecución del flujo de trabajo de GitHub y verás la aplicación en ejecución.

1.Selecciona **Acciones** y verás la configuración del flujo de trabajo antes de ejecutarse.

1. Selecciona el trabajo **Compilación y prueba de eShopOnWeb** en la sección **Todos los flujos de trabajo** de la página. 

1. El flujo de trabajo se compone de dos operaciones: **compilar y probar** e **implementar**. Puedes seleccionar cualquiera de las operaciones y ver su progreso o esperar hasta que se complete el trabajo.

1. Ve a Azure Portal [https://portal.azure.com](https://portal.azure.com) y ve al grupo de recursos **az2006-rg** creado anteriormente. Verás que la Acción de GitHub, mediante una plantilla de bicep, ha creado un plan de Azure App Service + App Service en el grupo de recursos. 

1. Selecciona el recurso de App Service (el nombre de aplicación único generado anteriormente) y después selecciona **Examinar** cerca de la parte superior de la página para ver la aplicación web implementada.

## Ejercicio 3: Limpieza de recursos

En este ejercicio, eliminarás los recursos creados anteriormente en el laboratorio.

1. Ve a Azure Portal [https://portal.azure.com](https://portal.azure.com) e inicia Cloud Shell. Selecciona la sesión de shell **Bash**.

1. Ejecuta el siguiente comando para eliminar el grupo de recursos `az2006-rg`. También quitarás el plan de App Service y la instancia de App Service.

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**Nota**: El comando se ejecuta de forma asíncrona (configurado con el parámetro `--no-wait`), por lo que, aunque puedes ejecutar otro comando de la CLI de Azure inmediatamente después en la misma sesión de Bash, los grupos de recursos tardarán unos minutos en quitarse.

## Revisar

En este laboratorio, aprenderás a implementar un flujo de trabajo de Acción de GitHub que implementa una aplicación web de Azure.
