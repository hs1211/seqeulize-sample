Sequelize
=========
Sequelize에서 필요한 내용에 대하여 간략히 설명하기 위하여 내용을 정리하였다.
우선 왜 ORM을 사용하고 어떻게 쓰는지 먼저 살펴보자. 그리고 포함한 소스 코드에서는 sequelize + sqlite를 통하여 실제 프로젝트로 어떻게 활용할지 확인해 보자.


ORM은 왜 사용할까?
------------------

- Granularity: 테이블 수보다 객체의 수가 많을 수 있음. 반드시 1:1은 아님
- Inheritance: RDB는 상속을 구현하지는 못함
- Identity: DB는 동일성 개념이 정확하지만, 객체에서는 a==b(identity) or a.equals(b)(equality) 2가지 동일성이 보장됨
- Association: 객체의 관계는 단방향이지만, DB는 양방향
- Data Navigation: 데이터를 탐색하는 방식이 객체는 하나씩 접근하지만, DB는 Join과 같은 방식으로 접근함 


연결 설정
-----------
설정을 위한 파일은 아래와 같은 구성된다.

1. Sequelize 연결 설정
2. Model 연결 설정
3. Relations 연결 설정


- app/config/db.js

.. code-block:: javascript

  const Sequelize = require('sequelize');
  const env = require('./env');


  const sequelize = new Sequelize(env.DATABASE, env.DATABASE_USERNAME, env.DATABASE_PASSWORD, {
    dialect: env.DATABASE_DIALECT,
    storage: env.DATABASE_STORAGE,
    define: {
      underscored: true
    }
  });

  // Connect all the models/tables in the database to a db object,
  //so everything is accessible via one object
  const db = {};

  db.Sequelize = Sequelize;
  db.sequelize = sequelize;

  //Models
  db.todos = require('../models/todos.js')(sequelize, Sequelize);

  //Relations
  // (no relations with only one table)

  module.exports = db;


Test Connection
--------------------

아래는 db접속에 대한 테스트를 위한 코드이다.

.. code-block:: javascript

  const sequelize = require('./db')

  sequelize
    .authenticate()
    .then(() => {
      console.log('Connection has been established successfully.');
    })
    .catch(err => {
      console.error('Unable to connect to the database:', err);
    });


First Model Test
--------------------

.. code-block:: javascript

  const User = sequelize.define('user', {
    firstName: {
      type: Sequelize.STRING
    },
    lastName: {
      type: Sequelize.STRING
    }
  });

  // force: true will drop the table if it already exists
  User.sync({force: true}).then(() => {
    // Table created
    return User.create({
      firstName: 'John',
      lastName: 'Hancock'
    });
  });


Basic Usage
---------------
- Raw Query

.. code-block:: javascript

  sequelize.query('your query', [, options])

  # argument with question mark
  sequelize
    .query(
      'SELECT * FROM projects WHERE status = ?',
      { raw: true, replacements: ['active']
    )
    .then(projects => {
      console.log(projects)
    })

  # keyword argument
  sequelize
    .query(
      'SELECT * FROM projects WHERE status = :status ',
      { raw: true, replacements: { status: 'active' } }
    )
    .then(projects => {
      console.log(projects)
    })



- Read replication

.. code-block:: javascript

  const sequelize = new Sequelize('database', null, null, {
  dialect: 'mysql',
  port: 3306
  replication: {
      read: [
        { host: '8.8.8.8', username: 'read-username', password: 'some-password' },
        { host: '9.9.9.9', username: 'another-username', password: null }
      ],
      write: { host: '1.1.1.1', username: 'write-username', password: 'any-password' }
    },
    pool: { // If you want to override the options used for the read/write pool you can do so here
      max: 20,
      idle: 30000
    },
  })

Model Definition
--------------------
모델과 테이블 사이의 매핑을 정의하기 위하여 define한다. 우선 아래 예제를 통하여 어떻게 사용하는지 확인하면 좋을 거 같아서 아래 예제를 먼저 살펴보기를 권한다.

- 예제

.. code-block:: text

  CREATE TABLE IF NOT EXISTS books (
    book_id SERIAL PRIMARY KEY,
    pub_id INTEGER REFERENCES publisher NOT NULL,
    title VARCHAR(64) NOT NULL,
    author VARCHAR(16) NOT NULL,
    stock SMALLINT NOT NULL DEFAULT 1,
    register_date TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
  );

.. code-block:: javascript

  sequelize.define('Books', {
      book_id: {type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true},
      pub_id: {type: DataTypes.INTEGER, allowNull: false, references: {model: models.Publisher, key: 'pub_id'}},
      title: {type: DataTypes.STRING(64), allowNull: false},
      author: {type: DataTypes.STRING(16), allowNull: false},
      stock: {type: DataTypes.INTEGER, defaultValue: 1},
      register_date: {type: DataTypes.DATE, defaultValue: DataTypes.NOW}
  }, {
      classMethods: {},
      tableName: 'books',
      freezeTableName: true,
      underscored: true,
      timestamps: false
  });

  db.Publisher.hasMany(db.Books, {foreignKey: 'pub_id'});

- 모델 사용방법 예시

.. code-block:: javascript

  const Project = sequelize.define('project', {
    title: Sequelize.STRING,
    description: Sequelize.TEXT
  })

  const Task = sequelize.define('task', {
    title: Sequelize.STRING,
    description: Sequelize.TEXT,
    deadline: Sequelize.DATE
  })

- Timestamp

  디폴트로 모델이 생성될때, createdAt 어트리뷰트와 updatedAt항목이 생성된다. 

.. code-block:: shell

  # 마이그레이션을 한다면 아래와 같이 되어야 한다
  module.exports = {
    up(queryInterface, Sequelize) {
      return queryInterface.createTable('my-table', {
        id: {
          type: Sequelize.INTEGER,
          primaryKey: true,
          autoIncrement: true,
        },

        // Timestamps
        createdAt: Sequelize.DATE,
        updatedAt: Sequelize.DATE,
      })
    },
    down(queryInterface, Sequelize) {
      return queryInterface.dropTable('my-table');
    },
  }

- Model options 

  모델 옵션을 통하여 getter/setter를 등록한 부분입니다.

.. code-block:: javascript

  const Foo = sequelize.define('foo', {
    firstname: Sequelize.STRING,
    lastname: Sequelize.STRING
  }, {
    getterMethods: {
      fullName() {
        return this.firstname + ' ' + this.lastname
      }
    },

    setterMethods: {
      fullName(value) {
        const names = value.split(' ');

        this.setDataValue('firstname', names.slice(0, -1).join(' '));
        this.setDataValue('lastname', names.slice(-1).join(' '));
      },
    }
  });

- validation

  모델 내부에 validation 로직을 추가하여 사용가능하며, validation로직은 'create', 'update' or 'save'에서 자동으로 호출한다.
    
.. code-block:: javascript

  const Pub = Sequelize.define('pub', {
    name: { type: Sequelize.STRING },
    address: { type: Sequelize.STRING },
    latitude: {
      type: Sequelize.INTEGER,
      allowNull: true,
      defaultValue: null,
      validate: { min: -90, max: 90 }
    },
    longitude: {
      type: Sequelize.INTEGER,
      allowNull: true,
      defaultValue: null,
      validate: { min: -180, max: 180 }
    },
  }, {
    validate: {
      bothCoordsOrNone() {
        if ((this.latitude === null) !== (this.longitude === null)) {
          throw new Error('Require either both latitude and longitude or neither')
        }
      }
    }
  })

- Configuration

  Timestamp: createdAt, updatedAt을 추가할지 결정(true: 추가, false: 안함)
  Paranoid: soft delete on and 'deletedAt'항목 추가
  underscored: attribute name 생성시 '_' 사용
  table_name: 직접 테이블 이름 명시
  version: optimistic locking을 enable시킴. 필드 업데이트 시 버전 정보 사용


Model Usage
---------------

- Usage

  가령 아래와 같은 쿼리가 있다면, 어떻게 변환이 가능할까? 아래를 살펴보자.

.. code-block:: TEXT

  SELECT *
  FROM `Projects`
  WHERE (
    `Projects`.`name` = 'a project'
    AND (`Projects`.`id` IN (1,2,3) OR `Projects`.`id` > 10)
  )
  LIMIT 1;
    
.. code-block:: text

  Project.findOne({
    where: {
      name: 'a project',
      id: {
        [Op.or]: [
          [1,2,3],
          { [Op.gt]: 10 }
        ]
      }
    }
  })
  
- Eager loading

  쿼리를 통하여 연관된 데이터도 함께 가져오기 위한 방법을 'eager loading'이라고 한다. 해당 아이디어는 find or findall과 같은 함수에서 
  'include'와 같은 어트리뷰트를 사용해서 활용할 수 있다.

- 모델 관계 정의

.. code-block:: text

  const User = sequelize.define('user', { name: Sequelize.STRING })
  const Task = sequelize.define('task', { name: Sequelize.STRING })
  const Tool = sequelize.define('tool', { name: Sequelize.STRING })

  Task.belongsTo(User)
  User.hasMany(Task)
  User.hasMany(Tool, { as: 'Instruments' })

  sequelize.sync().then(() => {
    // this is where we continue ...
  })

- 사용하는 코드

.. code-block:: text

  Task.findAll({ include: [ User ] }).then(tasks => {
    console.log(JSON.stringify(tasks))
  })

Query
---------
아래와 같이 모델의 호출여부 뿐아니라, 정렬을 포함하는 쿼리를 사용할 수 있다.

.. code-block: javascript

  Company.findAll({
    include: [ { model: Division, as: 'Div' } ],
    order: [ [ { model: Division, as: 'Div' }, 'name', 'DESC' ] ]
  });

- attribute

.. code-block:: javascript

  Model.findAll({
    attributes: ['foo', 'bar']
  });

  SELECT foo, bar ...

- where clause

.. code-block:: javascript

  Post.findAll({
    where: {
      authorId: 2
    }
  });
  // SELECT * FROM post WHERE authorId = 2

- Pagination / Limiting

.. code-block:: javascript

  // Fetch 10 instances/rows
  Project.findAll({ limit: 10 })

  // Skip 8 instances/rows
  Project.findAll({ offset: 8 })

  // Skip 5 instances and fetch the 5 after that
  Project.findAll({ offset: 5, limit: 5 })

Association
-------------
sequelize에 존재하는 다양한 타입을 설명한다. 

- One-To-One Association

  1:1관계는 정확하게 두 모델의 외래키가 1:1로 매핑되는 것을 의미한다.

- belongsTo

.. code-block:: javascript

  const Player = this.sequelize.define('player', {});
  const Team = this.sequelize.define('team', {});

  player.belongsTo(Team);

- Foreign key

  아래와 같이 외래키 명을 사용할 경우 두가지 경우로 자동생성이 가능하다. 만약 직접 이름을 지정하는 경우도 아래에 추가적으로 살펴보자.

.. code-block:: javascript

  # underscore 옵션이 없는 경우
  const User = this.sequelize.define('user', {/* attributes */})
  const Company  = this.sequelize.define('company', {/* attributes */});

  User.belongsTo(Company); // Will add companyId to user

  # 'underscore'옵션을 켜고 생성할 경우
  const User = this.sequelize.define('user', {/* attributes */}, {underscored: true})
  const Company  = this.sequelize.define('company', {
    uuid: {
      type: Sequelize.UUID,
      primaryKey: true
    }
  });

  User.belongsTo(Company); // Will add company_uuid to user

  # 명시적으로 컬럼명을 지정하는 경우
  const User = this.sequelize.define('user', {/* attributes */})
  const UserRole  = this.sequelize.define('userRole', {/* attributes */});

  User.belongsTo(UserRole, {as: 'role'}); // Adds roleId to user rather than userRoleId

- HasOne

  타겟 모델에 관계가 1:1인경우에 사용함

.. code-block:: javascript

  const User = sequelize.define('user', {/* ... */})
  const Project = sequelize.define('project', {/* ... */})

  // One-way associations
  Project.hasOne(User)


Transaction
-------------
2가지 종류의 트랜잭션을 지원한다. 

1) auto commit

자동으로 설정할 경우에는 다음과 같은 설정을 지정한다.

.. code-block:: javascript

  const cls = require('continuation-local-storage'),
  namespace = cls.createNamespace('my-very-own-namespace');

  # to enable
  const Sequelize = require('sequelize');
  Sequelize.useCLS(namespace);

  new Sequelize(....);


2) manual commit

직접 트랜잭션을 관리하기 위해서는 '{transaction:t}'를 두번째 아규먼트로 전달해야 한다.

.. code-block:: javascript

  return sequelize.transaction(function (t) {

    // chain all your queries here. make sure you return them.
    return User.create({
        firstName: 'Abraham',
        lastName: 'Lincoln'
      }, {transaction: t}).then(function (user) {
        return user.setShooter({
          firstName: 'John',
          lastName: 'Boothe'
        }, {transaction: t});
      });

  }).then(function (result) {
      // Transaction has been committed
      // result is whatever the result of the promise chain returned to the transaction callback
  }).catch(function (err) {
    // Transaction has been rolled back
    // err is whatever rejected the promise chain returned to the transaction callback
  });





Reference
---------
- http://docs.sequelizejs.com/manual/tutorial


