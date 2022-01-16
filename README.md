# Cliente de GitHub
## Creación de una extensión con GraphQL

## Jaime Simeón Palomar Blumenthal - alu0101228587@ull.edu.es

El objetivo de esta práctica es crear una extensión para el cliente de GitHub (gh) para la consola en GraphQl.


### **Índice**

1. [¿Por qué GraphQL?](#por-que-graphql)
2. [Estructura de una operación GraphQL](#estructura-graphql)
3. [Cómo insertar las consultas en un ejecutable JavaScript](#insertar-graphql)
4. [Cómo ejecutar las consultas GraphQL en JavaScript](#ejecutar-graphql)
5. [Tests con Mocha y Chai](#mocha-chai)
6. [Documentación con JSDOC](#jsdoc)


### **¿Por qué GraphQL?** <a name="por-que-graphql"/>

GraphQL es un lenguaje de consulta y modificación de la información de un servicio web. Es decir, sirve para que un programa se comunique con una página web.

A diferencia de REST, con el que al hacer la consulta obteníamos una página entera en formato JSON o XML, con este lenguaje sólo le solicitaremos al servicio web los objetos que queramos, sin necesidad de tener que investigar y averiguar dónde se sitúan.


### **Estructura de una operación GraphQL** <a name="estructura-graphql"/>

Una operación en GraphQL puede ser tanto de consulta como de modificación. A estas se las denominan _query_ y _mutation_ respectivamente. Tienen la siguiente estructura:

```
[query|mutation] [Nombre de la operación] (Posibles argumentos){
    Campos a extraer/modificar en la operación
}
```

La operación empieza indiccando si queremos hacer una _query_ o una _mutation_, podemos darle un nombre y pasarle argumentos si queremos. Dentro de la consulta especificaremos con qué campos trabajaremos. A su vez estos también pueden tener argumentos y otros campos. Veamos una consulta real:

```
query {
  organization(login: "ULL-ESIT-DMSI-1920") {
    membersWithRole(first: 20) {
      nodes {
        name
      }
    }
  }
}
```

Empezamos indicando que vamos a hacer un _query_, luego le indicamos que lo vamos a hacer sobre el objeto _organization_ que toma como parámetro un nombre de usuario u organización. Dentro de este consultaremos el objeto _membersWithRole_ y como parámetro le pasamos que nos muestre los 20 primeros nodos, y de estos el nombre.

Esta consulta devuelve una lista de nombres de usuario pertenecientes a una organización dada.


### **Cómo insertar las consultas en un ejecutable JavaScript** <a name="insertar-graphql"/>

En primer lugar, tendremos que generalizar las consultas, y las insertaremos en funciones flecha, y tomarán varios parámetros.

La primera consulta es la del apartado anterior pero generalizada para funcionar con cualquier organización:

```js
const getMemberList = (owner) => `
query {
  organization(login: "${owner}") {
    membersWithRole(first: 20) {
      nodes {
        name
      }
    }
  }
}
`;
```

La segunda devuelve una lista de repositorios dada una organización:

```js
const getRepoList = (owner) => `
query {
  organization(login: "${owner}") {
    repositories(first: 50) {
      nodes {
        name
      }
    }
  }
}
`;
```

La tercera devuelve la lista ficheros de un repositorio dado su nombre y organización:

```js
const getRepoContent = (owner, repo) => `
query {
  repository(owner: "${owner}", name: "${repo}") {
    object(expression: "master:") {
      ... on Tree {
        entries {
          name
        }
      }
    }
  }
}
`;
```

Y la última nos informará sobre qué ramas tiene un repositorio dado su nombre y organización:

```js
const getRepoBranches = (owner, repo) => `
query {
  repository(owner: "${owner}", name: "${repo}") {
    refs(first: 10, refPrefix: "refs/heads/") {
      nodes {
        name
      }
    }
  }
}
`;
```


### **Cómo ejecutar las consultas de GraphQL en JavaScript** <a name="ejecutar-graphql"/>

Para ejecutar nuestras consultas haremos uso de la librería _shelljs_ y del cliente de GitHub, gh.

En primer lugar, tenemos que copiar la clase _shelljs_ en una constante para utilizarla en nuestro programa:

```js
const shell = require('shelljs');
```

Y comprobar que _gh_ y _git_ están instalados:

```js
if (!shell.which('git')) {
	shell.echo("Git not installed.");
	shell.exit(1);
}

if (!shell.which('gh')) {
	shell.echo("gh not installed.");
	shell.exit(1);
}
```

Ahora sí, para ejecutar las consultas escribiremos lo siguiente:

```js
shell.exec(`gh api graphql --paginate -f query='${getRepoContent(owner, repoName)}' --jq '.data.repository.object.entries.[].name'`);
```

Estamos haciendo uso del comando _gh api_ con la opción para GraphQL. Le estamos indicando que no queremos forzar la consulta (_-f_) y en el query estamos insertando nuestra tercera consulta, a la que le estamos pasando dos variables que previamente hemos cargado con la organización y el nombre de un repositorio.

Esta consulta a secas nos devuelve un objeto JSON que aún debe ser filtrado con JQuery (_--jq_).


### **Tests con Mocha y Chai** <a name="mocha-chai"/>

Antes de publicar nuestro módulo debemos hacerle ciertos tests. En este caso se los haremos únicamente a las funciones exportadas desde el fichero _members.js_.

Para esto utilizaremos los módulos Mocha y Chai. Los añadimos a las _devDependencies_, los instalamos con _npm install_ y añadimos el script de tests:

```json
"scripts": {
    "test": "mocha --reporter spec"
}
```

Ahora, tendremos que escribir los tests en sí. Según la documentación de Chai debemos crear un directorio _test_ donde crearemos un fichero con el mismo nombre que el que exporta las funciones de nuestro proyecto (_members.js_ en nuestro caso).
Los tests unitarios tienen bloques descriptivos anidados unos dentro de otros. Si nos fijamos bien, no son más que funciones que toman como primer parámetro un string con una descripción breve, y como segundo parámetro una función que puede contener otro bloque descriptivo o un test en sí con la siguiente semántica:

```js
funcionExportada(argumentos...).should.equal(valor)
```

Esta semántica permite unos tests muy legibles. Podemos leer que "La _funciónExportada_ debería ser igual a _valor_". Chai permite otras comparaciones (desigualdad, por ejemplo), la negación, y valores predefinidos.

Finalmente, nuestros tests quedan de la siguiente forma:

```js
// Lista de miebros de la organización
const memberList = "Este string es muy largo";

// Lista de repositorios de la organización
const repoList = "Este string es aún más largo";

const reposAndMembers = memberList + repoList;

describe('Gets a:', function() {
    it('\t - List of organization members', function() {
        orgMembers("ULL-ESIT-DMSI-1920", null).should.equal(memberList);
    });

    it('\t - List of organization members and repositories', function() {
        orgMembers("ULL-ESIT-DMSI-1920", "-r").should.equal(reposAndMembers);
    });
});
```

De esta forma, si ejecutamos los tests con _npm test_ obtendremos la

```
> @alu0101228587/gh-members@1.1.4 test /home/usuario/Documents/gh-members-gql
> mocha --reporter spec

  Gets a:

-- Members of ULL-ESIT-DMSI-1920 --

Casiano Rodriguez-Leon
Casiano
Jaime Simeón Palomar Blumenthal
Antonella García
Sergio P
Laura Manzini

    ✔    - List of organization members (566ms)

-- Members of ULL-ESIT-DMSI-1920 --

Casiano Rodriguez-Leon
Casiano
Jaime Simeón Palomar Blumenthal
Antonella García
Sergio P
Laura Manzini

-- Repositories of ULL-ESIT-DMSI-1920 --

ull-esit-dmsi-1920.github.io
react-apollo
graphql-js
...
prueba-laura-funciona
gh-teresa-repo-rename
pruebaTeresa

    ✔    - List of organization members and repositories (1080ms)

  2 passing (2s)
```

Nuestros tests han tenido éxito. Sólo falta formalizar la documentación y la versión final de nuestro módulo estaría listo.


### **Documentación con JsDoc** <a name="jsdoc"/>

JsDoc es una herramienta muy cómoda que genera documentación estandarizada en formato web a partir de nuestros comentarios de código y _REDADME.md_. Para utilizarla debemos añadirla a nuestras _devDependencies_ e instalarla con _npm install_.

Hay que tener en cuenta que JsDoc requiere una sintaxis específica para que interprete correctamente nuestros comentarios.

```js
/** 
 * Returns list of members. If repo != null also a list of repositories
 * @param {string} orgName - Owner
 * @param {string} repo - If != null returns repositories
*/
function getOrgMembers(orgName, repo) {
    // ...
    // Código de la función
    // ...
}
```

El comentario se debe de situar justo encima de las funciones que lo requieran, y debe estar encorchetado por _/** Comentario */_. Cada línea debe empezar por un asterisco y podemos utilizar ciertas palabras clave como _@constructor_ o _@param_ para indicar que la función es el constructor, o qué argumentos toma. En este caso le hemos indicado a JsDoc tras una breve descripción que la función toma dos argumentos de tipo _string_ y una pequeña descripción de cada uno.

Tras haber hecho esto con todas las funciones de nuestro código que requieran una explicación, ejecutaremos _jsdoc members.js_. Esto crea un directorio _out_ con todos loos ficheros y código CSS, HTML y JavaScript de la web con nuestra documentación.


### **Pruebas de producción**

El objetivo de estas pruebas concretas serán que, cada vez que hagamos un push o un pull, se instalen todas las dependencias desde cero y se ejecuten los tests en un docker para comprobar que todo funciona correctamente.

En primer lugar, debemos crear nuestro fichero yaml en el directorio _.github/workflows/node.js.yml_, siendo _node.js.yml_ nuestro fichero. Este se obtuvo de una plantilla para nodeJS de GitHub, y contiene lo siguiente:

```yml
name: Node.js CI

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v2
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    - run: npm ci
    - run: npm test
      env:
        GITHUB_TOKEN: ${{secrets.GH_TOKEN}}
```

En primer lugar, en el bloque _on_ definimos qué acciones dispararán las instrucciones siguientes. En nuestro caso son los push y las solicitudes de pull en la rama master. Luego, en el bloque _jobs_ definiremos las acciones en sí.

En un docker con la última versión de ubuntu (_runs-on: ubuntu-latest_) se ejecutan las acciones definidas en el _tag_ V2 del repositorio con las acciones definidas por GitHub _actions/checkout_ para que un flujo de trabajo como el nuestro pueda acceder al docker. Después se instala la versión por defecto de Node, definida en la variable _matrix.node-version_. Finalmente se ejecutan los comandos para instalar las dependencias y ejecutar los tests (_npm ci_ y _npm test_). Además, como se accederá a nuestro repositorio desde una máquina externa, debemos proporcionarle un token de acceso de GitHub que previamente habremos definido en los secretos del repositorio. Esto lo hacemos estableciendo un valor para la variable de entorno _GITHUB_TOKEN_.