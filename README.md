**Problem:** We want to easily build features that show data and render graphs. In our setup, filtering and aggregating data sets in GraphQL is complicated and time-consuming.

**Solution:** Write React components that, using H1QL, query the database directly to get their data needs. H1QL is a safe subset of SQL and enforces strict access control to only expose data the requester can see. Engineers can quickly build reporting features as they are already familiar with the power of SQL and don’t have to invest in making it secure. H1QL is a typed language, this enables the component to know what type of graphs or other display options can be used.

# Usage

## In React
H1QL is a React component that requires a query that will be used to render its children.

```
<H1QL query=“SELECT COUNT(*) as count, MAX(id) as max FROM users”>
   ({ rows }) => {
      <p>
        Number of users: {rows[0].count}<br />
        Latest id: {rows[0].max}
      </p>
   }
</H1QL>
```

## Inner workings of H1QL
H1QL is a subset of SQL, it only supports operations that can be executed safely. For example, only non-mutative operations are supported and it doesn't allow operations that need direct file access. The H1QL engine will only return data the user is authorized to see. This is where H1QL really shines, restrictions on data access are centralized and any query can be executed safely no matter the source of the request. 

Rather than relying on authorization logic living in the database, H1QL will transform the query send to the database to include the authorization rules. Authorization logic can be allied to row level, column level, and even column*row level (although this will give you some interesting "what is `NULL`" problems).

For example, if we would query teams and the system only exposes visible teams, we would transfrom the requested query:
```
SELECT teams.id FROM teams;
```
To a query that includes the authorization rules:
```
SELECT teams.id FROM (SELECT * FROM teams WHERE visible = true) teams
```

As we guard every row and every columns, we can safely accept any SQL request. Next to do "boring" reads, this allows users to do advanced computational operations on the data they can access. For example, a user can: count, avg, max, generate time series, or any other (safe) SQL operation. 

## Our (PoC) implementation:
```
     +                H1QL Engine                                                       +
User |               +---------------------------------------------------+     Database |
     |               |                                                   |              |
     |               |  tokenization  transform   transform   to sql     |              |
     |  H1QL Query   |   & parsing    sql->h1ql  unsafe->safe            |  SQL Query   |
     | +-----------> |       +            +           +          +       | +----------> |
     |               |       |            |           |          |       |              |
     |               | +---> | +---+----> | +---+---> | +---+--+ | +-+-> |              |
     |               |       |     |      |     |     |     |    |   |   |              |
     |               |       +     |      +     |     +     |    +   |   |              |
     |               |       1     +      2     +     3     +    4   +   |              |
     |               |          SQL AST      (unsafe)     (safe)    SQL  |              |
     |               |                         AREL        AREL          |              |
     |               |                                                   |              |
     |               +---------------------------------------------------+              |
     |                                                                                  |
     |                                                                        Response  |
     | <------------------------------------------------------------------------------+ |
     |                                                                                  |
     +                                                                                  +                                     
```

*(1) tokenization & parsing*

Our setup uses [pg_query](https://github.com/lfittl/pg_query) to parse an incomming H1QL request. It accepts a SQL query and returns a Ruby respresentaion of the PostgreSQL AST, using [to_arel](https://github.com/mvgijssel/to_arel) we transform this AST into [Arel](https://github.com/rails/rails/tree/master/activerecord/lib/arel) which we use as intermediate storage between processes.

*(2) transform SQL to h1ql*

Using a [visitor pattern]([https://en.wikipedia.org/wiki/Visitor_pattern), we're creating a new AST that only contains attributes that are allowed in H1QL. If the algorithm stumbles upon an unsafe or unknown node, it will fail by raising an exception.

*(3) transform unsafe->safe*

From the limited insecure SQL to SQL that only allows access to data the user can see. This process is probably the most important but at the same time most complicated step in the engine. 

Using a visitor, we again visit every node in the Arel AST and verify what the access rules apply to this object. If we visit a node that has restricted accessibility, we'll replace it with a conditional node that includes these access rules. 

*(4) to_sql*

The last process is to transform the AST to SQL. As we use Arel as intermediate storage, this process is just a simple call to `to_sql`.

# Bonus feature - Using H1QL in Rails to maker everything safe!
Any query executed within an H1QL block will be automatically secured. Engineers don't have to worry about IDORs as all call to database are automatically transformed into safe to run database calls.
```lang=ruby
class SecretController < ApplicationController
  around_action :h1ql
  
  def index
    return Secret.all # 1
  end
  
  private
  
  def h1ql
    H1QL.new(requester: User.first) do
      yield
    end
  end
end
```


