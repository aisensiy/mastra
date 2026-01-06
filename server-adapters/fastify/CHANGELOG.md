# @mastra/fastify

## 0.1.0-beta.15

### Minor Changes

- feat: Add Fastify and Koa server adapters ([#11568](https://github.com/mastra-ai/mastra/pull/11568))

  Introduces two new server adapters for Mastra:
  - **@mastra/fastify**: Enables running Mastra applications on Fastify
  - **@mastra/koa**: Enables running Mastra applications on Koa

  Both adapters provide full MastraServerBase implementation including route registration, streaming responses, multipart uploads, auth middleware, and MCP transport support.

  ## Usage

  ### Fastify

  ```typescript
  import Fastify from 'fastify';
  import { MastraServer } from '@mastra/fastify';
  import { mastra } from './mastra';

  const app = Fastify();
  const server = new MastraServer({ app, mastra });

  await server.init();

  app.listen({ port: 4111 });
  ```

  ### Koa

  ```typescript
  import Koa from 'koa';
  import bodyParser from 'koa-bodyparser';
  import { MastraServer } from '@mastra/koa';
  import { mastra } from './mastra';

  const app = new Koa();
  app.use(bodyParser());

  const server = new MastraServer({ app, mastra });

  await server.init();

  app.listen(4111);
  ```

### Patch Changes

- Updated dependencies [[`d2d3e22`](https://github.com/mastra-ai/mastra/commit/d2d3e22a419ee243f8812a84e3453dd44365ecb0), [`bc72b52`](https://github.com/mastra-ai/mastra/commit/bc72b529ee4478fe89ecd85a8be47ce0127b82a0), [`05b8bee`](https://github.com/mastra-ai/mastra/commit/05b8bee9e50e6c2a4a2bf210eca25ee212ca24fa), [`c042bd0`](https://github.com/mastra-ai/mastra/commit/c042bd0b743e0e86199d0cb83344ca7690e34a9c), [`940a2b2`](https://github.com/mastra-ai/mastra/commit/940a2b27480626ed7e74f55806dcd2181c1dd0c2), [`e0941c3`](https://github.com/mastra-ai/mastra/commit/e0941c3d7fc75695d5d258e7008fd5d6e650800c), [`20e6f19`](https://github.com/mastra-ai/mastra/commit/20e6f1971d51d3ff6dd7accad8aaaae826d540ed), [`4f0b3c6`](https://github.com/mastra-ai/mastra/commit/4f0b3c66f196c06448487f680ccbb614d281e2f7), [`74c4f22`](https://github.com/mastra-ai/mastra/commit/74c4f22ed4c71e72598eacc346ba95cdbc00294f), [`81b6a8f`](https://github.com/mastra-ai/mastra/commit/81b6a8ff79f49a7549d15d66624ac1a0b8f5f971), [`e4d366a`](https://github.com/mastra-ai/mastra/commit/e4d366aeb500371dd4210d6aa8361a4c21d87034), [`a4f010b`](https://github.com/mastra-ai/mastra/commit/a4f010b22e4355a5fdee70a1fe0f6e4a692cc29e), [`5627a8c`](https://github.com/mastra-ai/mastra/commit/5627a8c6dc11fe3711b3fa7a6ffd6eb34100a306), [`251df45`](https://github.com/mastra-ai/mastra/commit/251df4531407dfa46d805feb40ff3fb49769f455), [`c2b9547`](https://github.com/mastra-ai/mastra/commit/c2b9547bf435f56339f23625a743b2147ab1c7a6), [`58e3931`](https://github.com/mastra-ai/mastra/commit/58e3931af9baa5921688566210f00fb0c10479fa), [`4fba91b`](https://github.com/mastra-ai/mastra/commit/4fba91bec7c95911dc28e369437596b152b04cd0), [`106c960`](https://github.com/mastra-ai/mastra/commit/106c960df5d110ec15ac8f45de8858597fb90ad5), [`12b0cc4`](https://github.com/mastra-ai/mastra/commit/12b0cc4077d886b1a552637dedb70a7ade93528c)]:
  - @mastra/core@1.0.0-beta.20
  - @mastra/server@1.0.0-beta.20

## 0.1.0-beta.14

### Minor Changes

- Initial release of Fastify server adapter
