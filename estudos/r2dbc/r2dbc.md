# [Access a database with Micronaut Data R2DBC](https://guides.micronaut.io/latest/micronaut-data-r2dbc-repository-maven-java.html)

``` java
package example.micronaut;

import jakarta.transaction.Transactional;
import jakarta.validation.constraints.NotBlank;
import example.micronaut.domain.Genre;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.r2dbc.annotation.R2dbcRepository;
import io.micronaut.data.repository.reactive.ReactorPageableRepository;
import reactor.core.publisher.Mono;

@R2dbcRepository(dialect = Dialect.MYSQL)
public interface GenreRepository extends ReactorPageableRepository<Genre, Long> {

    Mono<Genre> save(@NotBlank String name);

    @Transactional
    default Mono<Genre> saveWithException(@NotBlank String name) {
        return save(name)
            .then(Mono.error(new DataAccessException("test exception")));
    }

    Mono<Long> update(@Id long id, @NotBlank String name);
}

```


``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import java.net.URI;
import java.util.List;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;

import example.micronaut.domain.Genre;
import io.micronaut.data.model.Page;
import io.micronaut.data.model.Pageable;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.MutableHttpResponse;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Delete;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Put;
import io.micronaut.http.annotation.Status;
import reactor.core.publisher.Mono;

@Controller("/genres")
class GenreController {

    private final GenreRepository genreRepository;

    GenreController(GenreRepository genreRepository) {
        this.genreRepository = genreRepository;
    }

    @Get("/{id}")
    Mono<Genre> show(long id) {
        return genreRepository
                .findById(id);
    }

    @Put
    Mono<HttpResponse<?>> update(@Body @Valid GenreUpdateCommand command) {
        return genreRepository.update(command.getId(), command.getName())
                    .thenReturn(HttpResponse
                        .noContent()
                        .header(HttpHeaders.LOCATION, location(command.getId()).getPath()));
    }

    @Get("/list")
    Mono<List<Genre>> list(@Valid Pageable pageable) {
        return genreRepository.findAll(pageable)
                    .map(Page::getContent);
    }

    @Post
    Mono<HttpResponse<Genre>> save(@Body("name") @NotBlank String name) {
        return genreRepository.save(name)
                .map(GenreController::createdGenre);
    }

    @Post("/ex")
    Mono<MutableHttpResponse<Genre>> saveExceptions(@Body @NotBlank String name) {
        return genreRepository.saveWithException(name)
         .map(GenreController::createdGenre)
         .onErrorReturn(HttpResponse.noContent());
    }

    @Delete("/{id}")
    @Status(HttpStatus.NO_CONTENT)
    Mono<Void> delete(long id) {
        return genreRepository.deleteById(id)
            .then();
    }

    @NonNull
    private static MutableHttpResponse<Genre> createdGenre(@NonNull Genre genre) {
        return HttpResponse
                .created(genre)
                .headers(headers -> headers.location(location(genre.getId())));
    }

    private static URI location(Long id) {
        return URI.create("/genres/" + id);
    }

    private static URI location(Genre genre) {
        return location(genre.getId());
    }
}

```

1. A classe é definida como um controlador com a anotação @Controller mapeada para o caminho/genres.
2. Use injeção de construtor para injetar um bean do tipo GenreRepository.
3. Mapeia uma GET solicitação para /genres/{id}, que tenta mostrar um gênero. Isso ilustra o uso de uma variável de caminho de URL.
4. Retornar um Reactor vazio Mono quando o gênero não existe faz com que o framework Micronaut responda com 404 (não encontrado).
5. Mapeia uma PUT solicitação para /genres, que tenta atualizar um gênero.
6. Adiciona @Valida qualquer parâmetro do método que requeira validação. Use um POJO fornecido como carga JSON na solicitação para preencher o comando.
7. O Mono.thenReturn(..) método é usado para retornar uma NO_CONTENT resposta somente se a atualização for bem-sucedida. Cabeçalhos são facilmente adicionados pela HttpResponseAPI.
8. Mapeia uma GET solicitação para /genres/list, que retorna uma lista de gêneros.
9. Você pode vincular Pageablecomo um argumento de método do controlador. Confira os exemplos na seção de testes a seguir e leia as opções de configuração do Pageable . Por exemplo, você pode configurar o tamanho padrão da página com a propriedade de configuração micronaut.data.pageable.default-page-size.
10. Mapeia uma POST solicitação para /genres, que tenta salvar um gênero. Este exemplo usa o Mono.map(..) método para obter o resultado da chamada para save(String name)e mapeá-lo para uma CREATEDresposta HTTP se a operação de salvamento for bem-sucedida.
11. Mapeia uma POST solicitação para /ex, o que gera uma exceção. Este exemplo demonstra como lidar com exceções com Mono.onErrorReturn(..), que simplesmente retorna uma NO_CONTENT resposta se ocorrer um erro. Casos de uso mais complexos (como registrar a exceção) podem ser tratados com Mono.onErrorResume(..).
12. Mapeia uma DELETE solicitação para /genres/{id}, que tenta remover um gênero. Isso ilustra o uso de uma variável de caminho de URL. O Mono.then()método é chamado para descartar o resultado e retornar um valor vazio Monosomente se a chamada para deleteByIdfor bem-sucedida.
