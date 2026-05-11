
# core.ts
---
````typescript
const jsonFilePath = __dirname + '/data.temp.json'; // Caminho do arquivo JSON onde os dados serão armazenados

const list: string[] = await loadFromFile(); // Carrega a lista do arquivo ao iniciar a aplicação


// Função responsável por carregar os dados do arquivo
async function loadFromFile() {
  try {
    const file = Bun.file(jsonFilePath);  // Abre o arquivo usando Bun.file()

    const content = await file.text(); // Lê o conteúdo do arquivo como texto

    return JSON.parse(content) as string[]; // Converte o JSON em array de strings

  } catch (error: any) {

    // Caso o arquivo não exista, retorna um array vazio
    if (error.code === 'ENOENT')
      return [];

    throw error; // Qualquer outro erro será lançado
  }
}


// Função responsável por salvar os dados no arquivo
async function saveToFile() {
  try {

    await Bun.write(jsonFilePath, JSON.stringify(list)); // Converte a lista para JSON e salva no arquivo

  } catch (error: any) {

    throw new Error("Erro ao salvar os dados no arquivo: " + error.message); // Lança um erro personalizado caso falhe
  }
}


// Adiciona um novo item na lista
async function addItem(item: string) {

  list.push(item); // Adiciona o item no array

  await saveToFile(); // Salva as alterações no arquivo
}


// Retorna todos os itens da lista
async function getItems() {
  return list;
}


// Atualiza um item existente pelo índice
async function updateItem(index: number, newItem: string) {

  // Verifica se o índice é válido
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");

  list[index] = newItem;  // Atualiza o item

  await saveToFile(); // Salva as alterações
}


// Remove um item da lista pelo índice
async function removeItem(index: number) {

  // Verifica se o índice é válido
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");

  list.splice(index, 1);  // Remove o item do array

  await saveToFile(); // Salva as alterações
}


// Exporta as funções para serem utilizadas em outros arquivos
export default {
  addItem,
  getItems,
  updateItem,
  removeItem
};

````
---
# api.ts
---
````typescript
import todo from "./core.ts"; // Importa as funções do sistema de tarefas


// Cria o servidor usando Bun
const server = Bun.serve({
  port: 3000,

  // Rotas da API
  routes: {

    // ROTA /api/todo

    "/api/todo": {

      // Método GET
      // Retorna todos os itens
      GET: async () => {

        const items = await todo.getItems();  // Busca os itens

        return Response.json(items); // Retorna em formato JSON
      },


      // Método POST
      // Adiciona um novo item
      POST: async (req) => {

        const data = await req.json() as any;  // Converte o corpo da requisição em JSON

        const item = data.item || null;  // Obtém o item enviado

        // Verifica se o item foi enviado
        if (!item)
          return Response.json(
            'Por favor, forneça um item para adicionar.',
            { status: 400 }
          );

        await todo.addItem(item); // Adiciona o item

        return Response.json(data);  // Retorna os dados enviados
      },
    },


    // ROTA /api/todo/:index

    "/api/todo/:index": {

      // Método PUT
      // Atualiza um item existente
      PUT: async (req) => {

        const index = parseInt(req.params.index); // Converte o parâmetro para número

        // Verifica se é um número válido
        if (isNaN(index))
          return Response.json(
            'Índice inválido. Um número inteiro é esperado.',
            { status: 400 }
          );

        const data = await req.json() as any; // Obtém os dados enviados

        const newItem = data.newItem || null; // Obtém o novo valor do item

        // Verifica se foi enviado
        if (!newItem)
          return Response.json(
            'Por favor, forneça um novo item para atualizar.',
            { status: 400 }
          );

        try {

          await todo.updateItem(index, newItem); // Atualiza o item

          // Retorna mensagem de sucesso
          return Response.json(
            `Item no índice ${index} atualizado para "${newItem}".`
          );

        } catch (error: any) {

          return Response.json(error.message, { status: 400 });  // Retorna erro caso aconteça
        }
      },


      // Método DELETE
      // Remove um item da lista
      DELETE: async (req) => {

        const index = parseInt(req.params.index); // Converte o índice para número

        // Verifica se o índice é válido
        if (isNaN(index))
          return Response.json(
            'Índice inválido.',
            { status: 400 }
          );

        try {

          await todo.removeItem(index); // Remove o item

          // Retorna mensagem de sucesso
          return Response.json(
            `Item no índice ${index} removido com sucesso.`
          );

        } catch (error: any) {

          return Response.json(error.message, { status: 400 });  // Retorna erro caso aconteça
        }
      },
    },
  },


  // Servidor de arquivos estáticos

  async fetch(req) {

    const url = new URL(req.url);  // Obtém a URL da requisição

    const path = url.pathname; // Obtém o caminho da URL

    // Define qual arquivo será servido
    const filePath = (path === '/')
      ? './public/index.html'
      : `./public${path}`;

    const file = Bun.file(filePath); // Abre o arquivo

    // Verifica se o arquivo existe
    if (await file.exists()) {

      return new Response(file); // Retorna o arquivo
    }

    return new Response("Not Found", { status: 404 });  // Caso não exista, retorna 404
  },
});


console.log(`Server running at http://localhost:${server.port}`); // Exibe mensagem no terminal indicando que o servidor iniciou
````
---
