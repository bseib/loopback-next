---
lang: en
title: 'Migrating custom model methods'
keywords: LoopBack 4.0, LoopBack 4, LoopBack 3, Migration
sidebar: lb4_sidebar
permalink: /doc/en/lb4/migration-models-methods.html
---

## Introduction

In LoopBack 3, a model class has three responsibilities: a model describing shape of data, a repository providing data-access APIs, and a controller implementing REST API.

## Customizing data-access APIs in LB3

To override the behaviour of a [PersistedModel](https://apidocs.loopback.io/loopback/#persistedmodel) the developer has two options for writing **custom methods**:
- [Via server boot script](https://loopback.io/doc/en/lb3/Customizing-models.html#via-server-boot-script)
- [Via model's script](https://loopback.io/doc/en/lb3/Customizing-models.html#via-your-models-script)


The following LB3 code snippets show how to reimplement Note.find() to override the built-in [find()](https://apidocs.loopback.io/loopback/#persistedmodel-find) method.


**/server/boot/script.js**
```js
module.exports = function(app) {
  var Note = app.models.Note;
  var find = Note.find;
  var cache = {};

  Note.find = function(filter, cb) {
    var key = '';
    if(filter) {
      key = JSON.stringify(filter);
    }
    var cachedResults = cache[key];
    if(cachedResults) {
      console.log('serving from cache');
      process.nextTick(function() {
        cb(null, cachedResults);
      });
    } else {
      console.log('serving from db');
      find.call(Note, function(err, results) {
        if(!err) {
          cache[key] = results;
        }
        cb(err, results);
      });
    }
  }
}
```

OR

**common/models/Note.js**
```js
module.exports = function(Note) {
  Note.on('dataSourceAttached', function(obj){
    var find = Note.find;
    Note.find = function(filter, cb) {
      return find.apply(this, arguments);
    };
  });
};
```

In LoopBack 4, data-access APIs are implemented by [repositories](../../Repositories.md) that are decoupled from models.

A Repository represents a specialized Service interface that provides strong-typed data access (for example, CRUD) operations of a domain model against the underlying database or service. A single model can be used with multiple different Repositories. A Repository can be defined and implemented by application developers. LoopBack ships a few predefined Repository interfaces for typical CRUD and KV operations. These Repository implementations leverage Model definition and DataSource configuration to fulfill the logic for data access. See more examples at [Repository/CrudRepository/EntityRepository](https://github.com/strongloop/loopback-next/blob/master/packages/repository/src/repositories/repository.ts) and [KeyValueRepository](https://github.com/strongloop/loopback-next/blob/master/packages/repository/src/repositories/kv.repository.ts).

A CLI generated repository for a model `Note` would look like this:

```ts
export class NoteRepository extends DefaultCrudRepository<
  Note,
  typeof Note.prototype.id,
  NoteRelations
> {
  constructor(
    @inject('datasources.db') dataSource: DbDataSource,
  ) {
    super(Note, dataSource);
  }
}
```

It extends [DefaultCrudRepository](https://github.com/strongloop/loopback-next/blob/master/packages/repository/src/repositories/legacy-juggler-bridge.ts). To customize the `find` method, you would override this method in the `NoteRepository` class.


## Customizing REST APIs in LB3

In LoopBack 3 LoopBack models automatically have a standard set of HTTP endpoints that provide REST APIs for create, read, update, and delete (CRUD) operations on model data.

To control whether the REST API from is public or not, the developer has two options:
- specifying certain options in `server/model-config.json`
- calling `disableRemoteMethodByName()` from model's script file

See [Exposing and hiding models, methods, and endpoints](https://loopback.io/doc/en/lb3/Exposing-models-over-REST.html#exposing-and-hiding-models-methods-and-endpoints) for details.


**/server/model-config.json**

```json
"Note": {
  "public": true,
  "dataSource": "db"
},
```

This exposes all the REST API endpoints of the model.
(To “hide” all the REST API endpoints of the model, simply change public to false.)


**/server/model-config.json**
```json
"MyUser": {
  "public": true,
  "dataSource": "db",
  "options": {
    "remoting": {
      "sharedMethods": {
        "*": false,
        "login": true,
        "logout": true
      }
    }
  }
}
```
This hides all the REST API endpoints of the model, except `login and logout`.


**common/models/Note.js**
```js
Note.disableRemoteMethodByName('create');
Note.disableRemoteMethodByName('upsert');
Note.disableRemoteMethodByName('deleteById');
...
```
This disables REST API endpoints of the model by name.


In LoopBack 4, the REST APIs are implemented by [controllers](docs/site/Controllers.md) that are decoupled from models. A Controller is a class that implements operations defined by an application’s API. It implements an application’s business logic and acts as a bridge between the HTTP/REST API and domain/database models. A Controller operates only on processed input and abstractions of backend services / databases.

A CLI generated controller for a model `Note` and a repository `NoteRepository`would look like this:

```ts
export class NoteController {
  constructor(
    @repository(NoteRepository)
    public noteRepository : NoteRepository,
  ) {}

  @post('/notes', {
    responses: {
      '200': {
        description: 'Note model instance',
        content: {'application/json': {schema: getModelSchemaRef(Note)}},
      },
    },
  })
  async create(
    @requestBody({
      content: {
        'application/json': {
          schema: getModelSchemaRef(Note, {
            title: 'NewNote',
            exclude: ['id'],
          }),
        },
      },
    })
    note: Omit<Note, 'id'>,
  ): Promise<Note> {
    return this.noteRepository.create(note);
  }

  @get('/notes/count', {
    responses: {
      '200': {
        description: 'Note model count',
        content: {'application/json': {schema: CountSchema}},
      },
    },
  })
  async count(
    @param.query.object('where', getWhereSchemaFor(Note)) where?: Where<Note>,
  ): Promise<Count> {
    return this.noteRepository.count(where);
  }

  @get('/notes', {
    responses: {
      '200': {
        description: 'Array of Note model instances',
        content: {
          'application/json': {
            schema: {
              type: 'array',
              items: getModelSchemaRef(Note, {includeRelations: true}),
            },
          },
        },
      },
    },
  })
  async find(
    @param.query.object('filter', getFilterSchemaFor(Note)) filter?: Filter<Note>,
  ): Promise<Note[]> {
    return this.noteRepository.find(filter);
  }

  @patch('/notes', {
    responses: {
      '200': {
        description: 'Note PATCH success count',
        content: {'application/json': {schema: CountSchema}},
      },
    },
  })
  async updateAll(
    @requestBody({
      content: {
        'application/json': {
          schema: getModelSchemaRef(Note, {partial: true}),
        },
      },
    })
    note: Note,
    @param.query.object('where', getWhereSchemaFor(Note)) where?: Where<Note>,
  ): Promise<Count> {
    return this.noteRepository.updateAll(note, where);
  }

  @get('/notes/{id}', {
    responses: {
      '200': {
        description: 'Note model instance',
        content: {
          'application/json': {
            schema: getModelSchemaRef(Note, {includeRelations: true}),
          },
        },
      },
    },
  })
  async findById(
    @param.path.number('id') id: number,
    @param.query.object('filter', getFilterSchemaFor(Note)) filter?: Filter<Note>
  ): Promise<Note> {
    return this.noteRepository.findById(id, filter);
  }

  @patch('/notes/{id}', {
    responses: {
      '204': {
        description: 'Note PATCH success',
      },
    },
  })
  async updateById(
    @param.path.number('id') id: number,
    @requestBody({
      content: {
        'application/json': {
          schema: getModelSchemaRef(Note, {partial: true}),
        },
      },
    })
    note: Note,
  ): Promise<void> {
    await this.noteRepository.updateById(id, note);
  }

  @put('/notes/{id}', {
    responses: {
      '204': {
        description: 'Note PUT success',
      },
    },
  })
  async replaceById(
    @param.path.number('id') id: number,
    @requestBody() note: Note,
  ): Promise<void> {
    await this.noteRepository.replaceById(id, note);
  }

  @del('/notes/{id}', {
    responses: {
      '204': {
        description: 'Note DELETE success',
      },
    },
  })
  async deleteById(@param.path.number('id') id: number): Promise<void> {
    await this.noteRepository.deleteById(id);
  }
}

```

To hide a particular REST API endpoint, simply delete the appropriate function and its decorator.

To hide all REST API endpoints of a particular model, simply do not create a controller class for it.


## Rewriting a LoopBack 3 custom model method to a LoopBack 4 repository method

In progress...



