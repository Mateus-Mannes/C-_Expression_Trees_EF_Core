# Usando C# Expression Trees para criar uma consulta personalizada no banco de dados com EF Core

Usar o EF Core como ORM nos livra de ficar escrevendo e dando manutenção em querys SQL escritas diretamente no código fonte e da preocupação de traduzir os resultados das querys para as entidades dos nossos projetos. Porém, algumas vezes temos problemas em traduzir uma LINQ query que fazemos em um DBSet/Repository para a query SQL esperada, resultando em um erro de tradução como:

```bash
System.InvalidOperationException: The LINQ expression '...' could not be translated. Either rewrite the query in a form that can be translated, or switch to client evaluation explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync
```

E realmente, existem operações que são impossíveis de serem traduzidas pelo EF Core, mas o motivo do erro pode ser apenas uma má estruturação da arvore de expressões.

## Expression Trees

Uma Expression Trees é a representação de uma série de comandos ligados numa estrutura parecida com a de uma árvore. Ela precisa ser construída de baixo para cima e seus nodos são representados pela classe Expression no C#. Para clarificar, este é um exemplo de como podemos construir uma simples expressão lambda usando uma Expression Tree:

Lambda:

```C#
Func<int, bool> MenorQueCinco = n => n < 5;
```

Construindo Expression:

```C#
Func<int, bool> using System.Linq.Expressions;

var parametro = Expression.Parameter(typeof(int));

Expression<Func<int, bool>> MenorQueCincoExpression = 
    Expression.Lambda<Func<int, bool>>(
        Expression.LessThan(
            parametro,
            Expression.Constant(5)
        ), parametro
    );

var MenorQueCinco = MenorQueCincoExpression.Compile();
```

Por fim, essa seria a representação visual da arvore:

![image](https://user-images.githubusercontent.com/64140337/205742749-005f308a-37a2-4ab4-93c1-cf086dd59a2d.png)

Saber disso é importante pois, quando executamos algo do tipo:

```C#
var list = await _context.Entity.Where(x => x.Property == "value").ToListAsync();
```

O EF Core utiliza a estrutura da Expression Tree que representa o filtro "`x => x.Property == "value"`" para trabalhar como um compilador e gerar o código SQL adequado conforme vai passando nodo por nodo da arvore.

## Exemplificando com um caso real de uso

### O problema

Eu estava trabalhando com este tipo de query:

```C#
var result = await _context.Entity.Where(x => list.Contains(x.Property)).ToListAsync();
```

Que o EF Core traduz para:

```SQL
SELECT * FROM ENTITY WHERE PROPERTY IN ( ... )
```

Porém, o client do Oracle tem um limite de 1000 registros que podem ser filtrados no `IN`, que quando for ultrapassado resulta nesse tipo de erro:

```bash
OracleRelationalCommand.ExecuteReaderAsync() :  Oracle.ManagedDataAccess.Client.OracleException (0x80004005): ORA-01795: o número máximo de expressões em uma lista é de 1000
```

Uma possível solução é fazer o split da lista em listas de no máximo 1000 itens. Criando uma função para fazer esse split, ficaria mais ou menos assim:

```C#
var splitedList = SplitList(filter, 1000);
var result = new List<Entity>();
foreach (var list in splitedList)
    result.AddRange(await _context.Entity.Where(x => list.Contains(x.Property)).ToListAsync());
```

Dessa forma resolvemos o problema inicial. No entanto, caso seja necessário executar esse tipo query em outras partes do código para diferentes tabelas e filtrando por diferentes colunas, teremos que repetir essa mesma lógica com essas 5 linhas de código sempre. Portanto, a criação de um método genérico para esse tipo de consulta poderia melhorar o código.

Uma tentativa inicial criando um Extension Method ficaria dessa forma:

```C#
public async static Task<List<T>> SelectInChunksAsync<T, U>(this IQueryable<T> repository, List<U> listFilter, Func<T, U> selector)
{
    var splitedList = SplitList(listFilter, 1000);
    var result = new List<T>();
    foreach (var list in splitedList)
        result.AddRange(await repository.Where(x => list.Contains(selector(x))).ToListAsync());
    return result;
}
```

Note que o parâmetro `selector` é um lambda que retorna o valor da propriedade que eu quero filtrar. Executando esse método numa lista que já está na memória funciona normalmente. Porém, quando executamos ele num DbSet temos aquele erro de tradução:

```bash
System.InvalidOperationException: The LINQ expression '...' could not be translated. Either rewrite the query in a form that can be translated, or switch to client evaluation explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync
```

Isso acontece porque o lambda `selector` não consegue ser traduzido dessa forma, sendo inserido diretamente na Expression que é passada dentro do `Where()`

Logo, uma possível solução é construir a Expression que vai dentro do `Where()` do zero, usando a classe Expression, e construindo os parâmetros e operações na ordem correta, de forma legível para o EF Core:

```C#
public async static Task<List<T>> SelectInChunksAsync<T, U>(this IQueryable<T> repository, List<U> listFilter, Expression<Func<T, U>> selector)
{
    var splitedList = SplitList(listFilter, 1000);
    var result = new List<T>();
    foreach(var list in splitedList) 
        result.AddRange(await repository.SelectInAsync(list, selector));
    return result;
}

private async static Task<List<T>> SelectInAsync<T, U>(this IQueryable<T> repository, List<U> listFilter, Expression<Func<T, U>> selector)
{
    // Expressao constante pra lista de filtro
    var listaConstante = Expression.Constant(listFilter, listFilter.GetType());

    // parametro da Entidade na query
    var parameter = Expression.Parameter(typeof(T));

    // Expressao para o propriedade indicada no selector
    var memberExpression = (MemberExpression)selector.Body;
    var accessor = Expression.MakeMemberAccess(parameter, memberExpression.Member);

    // chamando o contains
    var callContains = Expression.Call(typeof(Enumerable), "Contains", new Type[] { accessor.Type }, listaConstante, accessor);
    var predicate = Expression.Lambda<Func<T, bool>>(callContains, parameter);

    return await repository.Where(predicate).ToListAsync();
}
```

Note que que agora estamos instanciando o parâmetros `selector` como `Expression<Func<T, U>>`, o que nos permite manipulá-lo como uma Expression Tree. Além disso, adicionamos o método `SelectInAsync`, que ficou responsável por construir a Expression Tree e executar e query no banco. Note também que trabalhamos com diversos tipos de Expressions, Constant, Parameter, Member, MethodCall (Contains), cada uma com a sua função na cadeia de comandos.

Debugando o código e analisando as variáveis, o valor do filtro final `predicate` ficará dessa forma:

```C#
{Param_0 => value(System.Collections.Generic.List`1[Type]).Contains(Param_0.Property)}
```

Portanto, agora temos uma forma genérica de fazer uma busca em partes, fazendo esse request:

```C#
var result = await _context.DbSet.SelectInChunksAsync(filterList, x => x.ColumnToFilter);
```

Representando isso no SQL, é como se tivéssemos diferentes listas de valores para filtrar no mesmo select, dessa forma:

```SQL
SELECT * FROM TABLE WHERE COLUMN IN ( ... )
UNION ALL 
SELECT * FROM TABLE WHERE COLUMN IN ( ... )
UNION ALL 
SELECT * FROM TABLE WHERE COLUMN IN ( ... )
...
```

Eu acredito que ter esse conhecimento sobre Expression Trees é uma carta na manga muito legal. Deixo aqui as referências que eu usei para escrever esse conteúdo:

- Documentação da Microsoft sobre Expression Trees:
    - [Building Expression Trees](https://learn.microsoft.com/en-us/dotnet/csharp/expression-trees-building)
    - [Expression Trees (C#)](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/)

- Palestra de Shay Rojansky sobre como o EF Core traduz as linq querys para SQL. Recomendo extremamente, muito interessante;
    - [Shay Rojansky - How Entity Framework translates LINQ all the way to SQL - Dotnetos Conference 2019](https://www.youtube.com/watch?v=r69ZxXgOIK4)
