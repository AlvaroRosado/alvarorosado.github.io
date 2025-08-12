---
title: El Problema de las Consultas N+1
date: 2025-08-11 17:00:00
categories: [Database, Doctrine, Symfony]
slug: el-problema-de-las-consultas-n-mas-1
tags: []
toc: true
---

El problema de las consultas N+1 es una ineficiencia común en aplicaciones que utilizan mapeo objeto-relacional (ORM). 
Este problema ocurre cuando una consulta inicial genera consultas adicionales por cada registro relacionado, lo que impacta negativamente el rendimiento a medida que crece el volumen de datos. 

**En este artículo nos centramos tanto ejemplos como soluciones en doctrine ORM, una herramienta popular en el ecosistema Symfony, pero los conceptos son aplicables a otros ORMs y bases de datos.**

> ⚠ **Aviso**: El código mostrado, está diseñado únicamente para explicar este problema concreto y sus soluciones.  
> No intentes copiarlo tal cual a producción — tu tech lead podría sufrir un infarto.  
> Si buscas buenas prácticas de arquitectura de software, tengo otros artículos dedicados a ello.


## Definición del Problema N+1

El problema N+1 se manifiesta al recuperar un conjunto de entidades y sus relaciones, generando una consulta inicial seguida de consultas adicionales por cada entidad relacionada. 
Por ejemplo, en una aplicación de blog que lista publicaciones y sus comentarios, un código no optimizado podría realizar:

1. Una consulta para obtener todas las publicaciones.
2. Una consulta por cada publicación para recuperar sus comentarios.

Consideremos el siguiente ejemplo en un controlador de Symfony:

```php
namespace App\Controller;

use App\Repository\PostRepository;
use Symfony\Component\HttpFoundation\JsonResponse;

class BlogController
{
    public function index(PostRepository $postRepository): JsonResponse
    {
        $posts = $postRepository->findAll();
        $result = array_map(
            fn($post) => [
                'title' => $post->getTitle(),
                'commentSummaries' => $post->getComments(),
            ],
            $posts
        );
        return new JsonResponse($result);
    }
}
```

Este código genera:

```sql
-- Consulta inicial para publicaciones
SELECT * FROM posts;

-- Consultas adicionales por cada publicación
SELECT * FROM comments WHERE post_id = 1;
SELECT * FROM comments WHERE post_id = 2;
SELECT * FROM comments WHERE post_id = 3;
-- ...y así sucesivamente
```

Con 100 publicaciones, se ejecutan 101 consultas: una inicial y una por cada publicación. Este patrón N+1 crea una carga significativa en la base de datos.

## Causas del Problema

El problema N+1 ocurre cuando las relaciones de entidades, configuradas por defecto como `LAZY` en Doctrine, se acceden dentro de un bucle, desencadenando consultas adicionales. Esto sucede al priorizar la simplicidad del código sobre la eficiencia de las consultas, un error común tanto en ORMs como en SQL puro.

## Identificación del Problema

Los indicadores del problema N+1 incluyen:

- Consultas ejecutadas dentro de un bucle en el código.
- Logs de Doctrine con múltiples consultas `SELECT` similares para diferentes identificadores.
- Degradación del rendimiento al aumentar el número de registros.
- Informes de herramientas de perfilado, como el Web Profiler de Symfony, que muestran consultas redundantes.

## Soluciones con Doctrine

Para mitigar el problema N+1, se deben optimizar las consultas para reducir el acceso a la base de datos. A continuación, se presentan varios enfoques:

### 1. Carga Anticipada (Eager Loading) con QueryBuilder

La carga anticipada permite recuperar entidades relacionadas en una sola consulta mediante un `JOIN`:

```php
// src/Repository/PostRepository.php
namespace App\Repository;

use App\Entity\Post;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

class PostRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Post::class);
    }

    /** @return Post[] */
    public function findAllWithComments(): array
    {
        return $this->createQueryBuilder('p')
            ->leftJoin('p.comments', 'c')
            ->addSelect('c')
            ->getQuery()
            ->getResult();
    }
}
```

```php
// src/Controller/BlogController.php
namespace App\Controller;

use App\Repository\PostRepository;
use Symfony\Component\HttpFoundation\JsonResponse;

class BlogController
{
    public function index(PostRepository $postRepository): JsonResponse
    {
        $posts = $postRepository->findAllWithComments();
        $result = array_map(
            fn($post) => [
                'title' => $post->getTitle(),
                'comments' => $post->getComments(),
            ],
            $posts
        );
        return new JsonResponse($result);
    }
}
```

Esto genera una sola consulta:

```sql
SELECT p.*, c.* FROM posts p LEFT JOIN comments c ON c.post_id = p.id;
```

### 2. Consulta Optimizada con IN

Cuando un `JOIN` no es ideal, se puede usar una consulta con `IN` para recuperar datos relacionados en un solo paso:

```php
// src/Repository/CommentRepository.php
namespace App\Repository;

use App\Entity\Comment;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

class CommentRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Comment::class);
    }

    /** @return Comment[] */
    public function findByPostIds(array $postIds): array
    {
        return $this->createQueryBuilder('c')
            ->where('c.post IN (:postIds)')
            ->setParameter('postIds', $postIds)
            ->getQuery()
            ->getResult();
    }
}
```

```php
// src/Controller/BlogController.php
namespace App\Controller;

use App\Entity\Post;
use App\Repository\CommentRepository;
use App\Repository\PostRepository;
use Symfony\Component\HttpFoundation\JsonResponse;

class BlogController
{
    public function index(PostRepository $postRepository, CommentRepository $commentRepository): JsonResponse
    {
        $posts = $postRepository->findAll();
        $postIds = array_map(fn(Post $post) => $post->getId(), $posts);

        $comments = $commentRepository->findByPostIds($postIds);
        $result = [];
        foreach ($posts as $post) {
            $result[] = [
                'title' => $post->getTitle(),
                'comments' => $comments,
            ];
        }
        return new JsonResponse($result);
    }
}
```

Esto genera dos consultas:

```sql
SELECT * FROM posts;
SELECT * FROM comments WHERE post_id IN (1, 2, 3, 4, 5);
```

**Este enfoque requiere organizar los datos en memoria, por lo que puede ser menos eficiente, pero evita el problema N+1 al reducir el número total de consultas a la base de datos.**

### 3. Consultas Directas sin ORM

En algunos casos, especialmente para vistas que solo requieren datos específicos y no necesitan la estructura completa de las entidades, realizar consultas SQL directas (sin usar el ORM) puede ser más eficiente. Esto evita la sobrecarga del mapeo objeto-relacional y permite optimizar la consulta para obtener exactamente los datos necesarios.
Por ejemplo, si solo necesitas mostrar un resumen de publicaciones y el número de comentarios por publicación en una vista, puedes escribir una consulta SQL directa:

```php
namespace App\Repository;

use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;
use Doctrine\DBAL\Connection;

class PostRepository extends ServiceEntityRepository
{

    public function __construct(ManagerRegistry $registry, private Connection $connection)
    {
        parent::__construct($registry, Post::class);
    }

    public function findAllWithCommentCount(): array
    {
        $sql = '
            SELECT p.id, p.title, COUNT(c.id) as comment_count
            FROM posts p
            LEFT JOIN comments c ON c.post_id = p.id
            GROUP BY p.id, p.title
        ';
        return $this->connection->executeQuery($sql)->fetchAllAssociative();
    }
}
```
```php
namespace App\Controller;

use App\Repository\PostRepository;
use Symfony\Component\HttpFoundation\JsonResponse;

class BlogController
{
    public function index(PostRepository $postRepository): JsonResponse
    {
        $posts = $postRepository->findAllWithCommentCount();
        $result = array_map(
            fn($post) => [
                'title' => $post['title'],
                'commentCount' => (int)$post['comment_count'],
            ],
            $posts
        );
        return new JsonResponse($result);
    }
}
```

Esto genera una única consulta SQL que cuenta los comentarios por publicación:

```sql
SELECT p.id, p.title, COUNT(c.id) as comment_count
FROM posts p
LEFT JOIN comments c ON c.post_id = p.id
GROUP BY p.id, p.title;
```

Ventajas:

* Reduce el número de consultas a una sola.
* Evita la carga de entidades completas, lo que disminuye el uso de memoria.
* Ideal para vistas que solo necesitan datos específicos (por ejemplo, listas o resúmenes).

Consideraciones:

* Pierdes las ventajas del ORM, como la validación automática y la gestión de relaciones.
* Requiere cuidado al manejar la lógica de negocio fuera del modelo de entidades.

### 4. Definir Relaciones como EAGER
En Doctrine, las relaciones entre entidades son configuradas por defecto como LAZY, lo que provoca el problema N+1 al acceder a las relaciones en un bucle. 
Una solución alternativa es definir las relaciones como **EAGER** en las anotaciones de las entidades, asegurando que las entidades relacionadas se carguen automáticamente en la consulta inicial.

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;

#[ORM\Entity(repositoryClass: "App\Repository\PostRepository")]
class Post
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: "integer")]
    private int $id;

    #[ORM\Column(type: "string")]
    private string $title;

    #[ORM\OneToMany(targetEntity: "App\Entity\Comment", mappedBy: "post", fetch: "EAGER")]
    private Collection $comments;

    public function __construct()
    {
        $this->comments = new ArrayCollection();
    }

    public function getId(): int
    {
        return $this->id;
    }

    public function getTitle(): string
    {
        return $this->title;
    }

    public function getComments(): Collection
    {
        return $this->comments;
    }
}

```

El controlador utiliza el método findAll() del repositorio para recuperar las publicaciones, y los comentarios se cargan automáticamente debido a la configuración EAGER.

```php
namespace App\Controller;

use App\Repository\PostRepository;
use Symfony\Component\HttpFoundation\JsonResponse;

class BlogController
{
    public function index(PostRepository $postRepository): JsonResponse
    {
        $posts = $postRepository->findAll();
        $result = array_map(
            fn($post) => [
                'title' => $post->getTitle(),
                'comments' => $post->getComments(),
            ],
            $posts
        );
        return new JsonResponse($result);
    }
}

```

La configuración EAGER genera una sola consulta que incluye tanto las publicaciones como sus comentarios asociados mediante un LEFT JOIN:
```sql
SELECT p.*, c.* FROM posts p LEFT JOIN comments c ON c.post_id = p.id;
```

Ventajas:

* Simplifica el código, ya que no necesitas modificar las consultas en el repositorio.
* Asegura que las relaciones se carguen automáticamente, evitando el problema N+1 sin intervención adicional.

Consideraciones:

* Puede cargar más datos de los necesarios si las relaciones no siempre se usan.
* Incrementa el consumo de memoria, especialmente si las relaciones contienen muchos registros.
* No es adecuado para todas las relaciones; debe usarse con precaución y solo en casos donde la carga de datos relacionados sea casi siempre necesaria.

## Importancia del Problema

El problema N+1 puede pasar desapercibido en aplicaciones con pocos datos, pero se vuelve crítico en sistemas con gran volumen de información. 
La sobrecarga de consultas puede ralentizar significativamente la aplicación, afectando la experiencia del usuario y la escalabilidad. 
Abordarlo desde el inicio asegura un rendimiento óptimo a largo plazo.


