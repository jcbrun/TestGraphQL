In the last two months, I have been working on a project which I used GraphQL to send data to the client-side. So I thought of writing a post on how I did this.
GraphQL is a query language for APIs and a runtime for fulfilling those queries with your existing data. GraphQL provides a complete and understandable description of the data in your API, gives clients the power to ask for exactly what they need and nothing more, makes it easier to evolve APIs over time, and enables powerful developer tools.
You can read more about GraphQL here
So let us get to implementing it in your flask project.
# Setting up your project
First lets setup your flask projects. Create a directory

    $ mkdir flask-graphql-project
    $ cd flask-graphql-project
Now create a virtual environment. This helps keep your project dependencies in one place by giving you a custom python installation. If you don’t have it installed already, install it.

    $ pip install virtualenv
Now create a virtual environment for your project and activate the virtualenv.

    $ virtualenv venv
    $ source venv/bin/activate
This creates a virtual environment inside your project folder. Then install your project dependencies.

    $ pip install flask flask-graphql flask-migrate flask-sqlalchemy graphene graphene-sqlalchemy
Now create an app.py file in your folder root directory. Add the following code to the file

    # Imports
    from flask import Flask
    # app initialization
    app = Flask(__name__)
    app.debug = True
    # Configs
    # TO-DO
    # Modules
    # TO-DO
    # Models
    # TO-DO
    # Schema Objects
    # TO-DO
    # Routes
    # TO-DO
    @app.route('/')
    def index():
        return '<p> Hello World</p>'
    if __name__ == '__main__':
         app.run()
Note, the to-do comments will be filled as we progress.
Now from the terminal run python app.py runserver this will set up a debug server, then visit 127.0.0.1:5000 If you see hello world written on the page then you’re good to go.
# Adding a database
Now, let's add a database to our application. We’re going to be using Sqlalchemy to create our DB models and SQLite for our demo DB.
Add the following code to app.py file.

    # Imports
    ...
    from flask_sqlalchemy import SQLAlchemy
    import os
    basedir = os.path.abspath(os.path.dirname(__file__))
    ...
    # Configs
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' +    os.path.join(basedir, 'data.sqlite')
    app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN'] = True
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
    # Modules
    db = SQLAlchemy(app)
    ...
Now let's create our models. Add the following code to app.py:

    ...
    # Models
    class User(db.Model):
        __tablename__ = 'users'
        uuid = db.Column(db.Integer, primary_key=True)
        username = db.Column(db.String(256), index=True, unique=True)
        posts = db.relationship('Post', backref='author')
        
        def __repr__(self):
            return '<User %r>' % self.username
    class Post(db.Model):
        __tablename__ = 'posts'
        uuid = db.Column(db.Integer, primary_key=True)
        title = db.Column(db.String(256), index=True)
        body = db.Column(db.Text)
        author_id = db.Column(db.Integer, db.ForeignKey('users.uuid'))
        def __repr__(self):
            return '<Post %r>' % self.title
...
In the code above we created two models Posts and User and created a relationship between the two. Now let's create some demo data.

    $ python
    >>> from app import db, User, Post
    >>> db.create_all()
    >>> john = User(username='johndoe')
    >>> post = Post()
    >>> post.title = "Hello World"
    >>> post.body = "This is the first post"
    >>> post.author = john
    >>> db.session.add(post)
    >>> db.session.add(john)
    >>> db.session.commit()
    >>> User.query.all()
    [<User 'johndoe'>]
    >>> Post.query.all()
    [<Post 'Hello World'>
# Adding GraphQL
Now let’s get to business.
## Schemas
GraphQL presents your objects to the world as a graph structure rather than a more hierarchical structure to which you may be accustomed. So we need to show which type of object will be shown in the graph. That’s where schemas come in. Schemas are used to describe your data. So let's create a schema for our models.
Add the following content to app.py:

    # Imports
    ...
    import graphene
    from graphene_sqlalchemy import SQLAlchemyObjectType, SQLAlchemyConnectionField
    ...
    # Schema Objects
    class PostObject(SQLAlchemyObjectType):
        class Meta:
            model = Post
            interfaces = (graphene.relay.Node, )
    class UserObject(SQLAlchemyObjectType):
       class Meta:
           model = User
           interfaces = (graphene.relay.Node, )
    class Query(graphene.ObjectType):
        node = graphene.relay.Node.Field()
        all_posts = SQLAlchemyConnectionField(PostObject)
        all_users = SQLAlchemyConnectionField(UserObject)
    schema = graphene.Schema(query=Query)
    ...
Now add the route to check out our GraphQL interface

    # Imports
    ...
    from flask_graphql import GraphQLView
    ...
    # Routes
    ...
    app.add_url_rule(
        '/graphql',
        view_func=GraphQLView.as_view(
            'graphql',
            schema=schema,
            graphiql=True # for having the GraphiQL interface
        )
    )
    ...
Now if you navigate to 127.0.0.1:5000/graphql you’ll see the GraphQLi view. Write your GraphQL query

    {
      allPosts{
        edges{
          node{
            title
            body
            author{
              username
            }
          }
        }
      }
    }
## Mutations

To be able to create posts and users you’ll need mutations. So let's add that to our app.
    ...
    # Schema Objects
    ...
    class CreatePost(graphene.Mutation):
        class Arguments:
            title = graphene.String(required=True)
            body = graphene.String(required=True) 
            username = graphene.String(required=True)
        post = graphene.Field(lambda: PostObject)
        def mutate(self, info, title, body, username):
            user = User.query.filter_by(username=username).first()
            post = Post(title=title, body=body)
            if user is not None:
                post.author = user
            db.session.add(post)
            db.session.commit()
            return CreatePost(post=post)
    class Mutation(graphene.ObjectType):
        create_post = CreatePost.Field()
    schema = graphene.Schema(query=Query, mutation=Mutation)
Now, let's add a new post with GraphQL mutations:
    mutation {
      createPost(username:"johndoe", title:"Hello 2", body:"Hello body 2"){
        post{
          title
          body
          author{
            username
          }
        }
      }
    }
