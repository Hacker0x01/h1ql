**Problem:** We want to quickly build features that show data and render graphs. In our setup, filtering and aggregating data sets in GraphQL is complicated and time-consuming.

**Solution:** Write React components that, using H1QL, query the database directly to get their data needs. H1QL is a safe subset of SQL and enforces strict access control to only expose data the requester is authorized to see. Engineers can quickly build reporting features as they are already familiar with the power of SQL and don’t have to invest in making it secure. H1QL is a typed language; this enables the component to know what type of graphs or other display options can be used.

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
H1QL is a subset of SQL; it only supports operations that can be executed safely. For example, only non-mutative operations are supported and it doesn't allow execution of operations that require direct file access. 

Rather than relying on authorization logic living in the database, H1QL will transform the query that will be sent to the database and includes the authorization rules. The authorization constraints can be applied to row level, column level, and even column*row level (although this will give you some interesting "what is `NULL`" problems).

For example, if we would query teams and the system only exposes visible teams, H1QL would transform the requested query:
```
SELECT teams.id FROM teams;
```
Into a query that includes the authorization rules:
```
SELECT teams.id FROM (SELECT * FROM teams WHERE visible = true) teams
```

As access to every row and every column is guarded, we can safely accept any SQL request no matter the source of the request. Next to "boring" reads, this allows users to do advanced computational operations on the data they can access. For example, a user can: count, avg, max, generate time series, or any other (safe) SQL operation. 

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

Our setup uses [pg_query](https://github.com/lfittl/pg_query) to parse an incoming H1QL request. It accepts a SQL query and returns a Ruby representation of the PostgreSQL AST, using [to_arel](https://github.com/mvgijssel/to_arel) we transform this AST into [Arel](https://github.com/rails/rails/tree/master/activerecord/lib/arel) which we use as intermediate storage between processes.

*(2) transform SQL to h1ql*

Using a [visitor pattern]([https://en.wikipedia.org/wiki/Visitor_pattern), we're creating a new AST that only contains attributes that are allowed in H1QL. If the algorithm stumbles upon an unsafe or unknown node, it will fail by raising an exception.

*(3) transform unsafe->safe*

This processor will transform the insecure Arel AST to an Arel AST that includes the authorization contraints. Using a visitor, we again visit every node in the Arel AST and verify what the access rules apply to this object. If we visit a node that has restricted accessibility, we'll replace it with a conditional node that includes these access rules. 

*(4) to_sql*

The last process is to transform the AST to SQL. As we use Arel as intermediate storage, this process is just a simple call to `to_sql`.

# Bonus feature - Using H1QL in Rails to maker everything safe!
Any query executed within an H1QL block will be automatically secured. Engineers don't have to worry about IDORs as all calls to the database are automatically secured.
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


