# Cliente de GitHub
## Creación de una extensión con GraphQL

## Jaime Simeón Palomar Blumenthal - alu0101228587@ull.edu.es

El objetivo de esta práctica es crear una extensión para el cliente de GitHub (gh) para la consola en GraphQl.


### **Índice**

1. [¿Por qué GraphQL?](#por-que-graphql)
2. [Estructura de una operación GraphQL](#estructura-graphql)
3. [Cómo insertar las consultas en un ejecutable JavaScript](#insertar-graphql)
4. [Cómo ejecutar las consultas GraphQL en JavaScript](#ejecutar-graphql)


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