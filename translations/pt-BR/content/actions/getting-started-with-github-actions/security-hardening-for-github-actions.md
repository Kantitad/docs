---
title: Fortalecimento de segurança para o GitHub Actions
shortTitle: Fortalecimento de segurança
intro: 'Boas práticas de segurança para usar recursos do {% data variables.product.prodname_actions %}.'
product: '{% data reusables.gated-features.actions %}'
versions:
  free-pro-team: '*'
  enterprise-server: '>=2.22'
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

Este guia explica como configurar o fortalecimento de segurança para certos recursos de {% data variables.product.prodname_actions %}. Se os conceitos do {% data variables.product.prodname_actions %} forem desconhecidos, consulte "[Principais conceitos para o GitHub Actions](/actions/getting-started-with-github-actions/core-concepts-for-github-actions)

### Usar segredos

Valores sensíveis nunca devem ser armazenados como texto simples em arquivos de fluxo de trabalho, mas como segredos. [Os segredos](/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets) podem ser configurados no nível da organização ou do repositório e permitem que você armazene informações confidenciais em {% data variables.product.product_name %}.

Os segredos usam [caixas fechadas de Libsodium](https://libsodium.gitbook.io/doc/public-key_cryptography/sealed_boxes) de modo que sejam criptografadas antes de atingir {% data variables.product.product_name %}. Isso ocorre quando o segredo é enviado [usando a interface de usuário](/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets-for-a-repository) ou através da [API REST](/rest/reference/actions#secrets). Esta criptografia do lado do cliente ajuda a minimizar os riscos relacionados ao registro acidental (por exemplo, registros de exceções e de solicitação, entre outros) dentro da infraestrutura do {% data variables.product.product_name %}. Uma vez realizado o upload do segredo, o {% data variables.product.product_name %} poderá descriptografá-lo para que possa ser injetado no tempo de execução do fluxo de trabalho.

Para ajudar a prevenir a divulgação acidental, o {% data variables.product.product_name %} usa um mecanismo que tenta redigir quaisquer segredos que aparecem nos registros de execução. Esta redação procura correspondências exatas de quaisquer segredos configurados, bem como codificações comuns dos valores, como Base64. No entanto, como há várias maneiras de transformar o valor de um segredo, essa anulação não é garantida. Como resultado, existem certas etapas proativas e boas práticas que você deve seguir para ajudar a garantir que os segredos sejam editados, e para limitar outros riscos associados aos segredos:

- **Nunca usar dados estruturados como um segredo**
    - Os dados não estruturados podem fazer com que ocorra uma falha na redação secreta nos registros, porque a redação depende, em grande parte, de encontrar uma correspondência exata para o valor específico do segredo. Por exemplo, não use um blob de JSON, XML, ou YAML (ou similar) para encapsular o valor de um segredo, já que isso reduz significativamente a probabilidade de os segredos serem devidamente redigidos. Em vez disso, crie segredos individuais para cada valor sensível.
- **Registre todos os segredos usados nos fluxos de trabalho**
    - Se um segredo for usado para gerar outro valor sensível dentro de um fluxo de trabalho, esse valor gerado deve ser formalmente [registrado como um segredo](https://github.com/actions/toolkit/tree/master/packages/core#setting-a-secret) para que seja reproduzido se alguma vez aparecer nos registros. Por exemplo, se, ao usar uma chave privada para gerar um JWT assinado para acessar uma API web, certifique-se de registrar que JWT é um segredo ou não será redigido se entrar na saída de do registro.
    - O registro de segredos também aplica-se a qualquer tipo de transformação/codificação. Se seu segredo foi transformado de alguma forma (como Base64 ou URL codificada), certifique-se de registrar o novo valor como um segredo também.
- **Audite como segredos são tratados**
    - Audite como os segredos são usados, para ajudar a garantir que estejam sendo tratados conforme o esperado. Você pode fazer isso revisando o código-fonte do repositório que executa o fluxo de trabalho e verificando quaisquer ações usadas no fluxo de trabalho. Por exemplo, verifique se eles não são enviados para hosts não pretendidos, ou impressos explicitamente na saída de um registro.
    - Visualize os registros de execução do seu fluxo de trabalho depois de testar entradas válidas/inválidas e, em seguida, verifique se os segredos estão sendo editados corretamente ou não são mostrados. Nem sempre é sempre óbvio como um comando ou ferramenta que você está invocando irá enviar erros para `STDOUT` e `STDERR`, e os segredos podem depois acabar em registros de erro. Como resultado, considera-se uma boa prática rever manualmente os registros do fluxo de trabalho depois de testar entradas válidas e inválidas.
- **Use as credenciais que tenham escopos mínimos**
    - Certifique-se de que as credenciais usadas nos fluxos de trabalho têm o menor privilégio necessário e esteja ciente de que qualquer usuário com acesso de gravação ao repositório terá acesso de leitura a todos os segredos configurados no seu repositório.
- **Audite e gire os segredos registrados**
    - Reveja, periodicamente, os segredos registrados para confirmar se ainda são necessários. Remova aqueles que não são mais necessários.
    - Gire os segredos periodicamente para reduzir a janela de tempo durante a qual um segredo comprometido é válido.

### Usando ações de terceiros

Os trabalhos individuais em fluxo de trabalho podem interagir com (e comprometer) outros trabalhos. Por exemplo, um trabalho que consulta as variáveis de ambiente usadas por um trabalho posterior, que escreve arquivos para um diretório compartilhado que um trabalho posterior processa, ou ainda mais diretamente, que interage com o conector do Docker e inspeciona outros contêineres em execução e executa comandos neles.

Isso significa que comprometer uma única ação dentro de um fluxo de trabalho pode ser muito significativo, uma vez que essa ação comprometida teria acesso a todos os segredos configurados no seu repositório e pode usar o `GITHUB_TOKEN` para gravar no repositório. Consequentemente, há um risco significativo em fornecer de ações de repositórios de terceiros no {% data variables.product.prodname_dotcom %}. Você pode ajudar a mitigar esse risco seguindo estas boas práticas:

* **Fixe as ações para um commit SHA de comprimento completo**

  Fixar uma ação para um commit SHA de comprimento completo é, atualmente, a única maneira de usar uma ação como uma versão imutável. Fixar um SHA em particular ajuda a mitigar o risco de um ator malicioso adicionar uma porta traseira ao repositório da ação, porque precisariam gerar uma colisão de SHA-1 para uma carga válida do objeto de Git.

  {% warning %}

  **Aviso:** A versão curta do commit SHA é insegura e nunca deve ser usada para especificar a referência do Git de uma ação. Devido ao modo como funcionam as redes de repositório, qualquer usuário pode bifurcar o repositório e fazer push de um commit criado que colida com o SHA curto. Isso faz com que os clones subsequentes falhem nesse SHA, pois se converte em um commit ambíguo. Como resultado, todos os fluxos de trabalho que usam o SHA encurtado falharão imediatamente.

  {% endwarning %}
* **Audite o código-fonte da ação**

  Certifique-se de que a ação está tratando o conteúdo do seu repositório e os segredos, como esperado. Por exemplo, verifique se os segredos não são enviados para os hosts não intencionais, ou se não são registrados inadvertidamente.

* **Fixe ações em uma etiqueta apenas se confiar no criador**

  Embora a fixação de um commit de SHA seja a opção mais segura, especificar uma etiqueta é a opção mais conveniente, além de ser amplamente usada. Se você desejar de especificar uma etiqueta, certifique-se de que você confia nos criadores da ação. O selo "Criador verificado" em {% data variables.product.prodname_marketplace %} é um sinal útil, já que indica que a ação foi escrita por uma equipe cuja identidade foi verificada por {% data variables.product.prodname_dotcom %}. Observe que há risco para esta abordagem, mesmo que você confie no autor, porque uma etiqueta pode ser movida ou excluída se um ator malicioso obtiver acesso ao repositório que armazena a ação.

### Considerar acesso entre repositórios

O {% data variables.product.product_name %} tem um escopo intencional para um único repositório por vez. O `GITHUB_TOKEN` usado no ambiente de fluxo de trabalho concede o mesmo nível de acesso que um usuário com acesso de gravação, porque qualquer usuário com acesso de gravação pode acessar esse token criando ou modificando arquivos de fluxo de trabalho. Os usuários têm permissões específicas para cada repositório. Portanto, fazer com que o `GITHUB_TOKEN` para um repositório conceda acesso a outro impactaria o modelo de permissão {% data variables.product.prodname_dotcom %} se não for implementado cuidadosamente. Da mesma forma, deve-se ter cuidado ao adicionar tokens de autenticação do {% data variables.product.prodname_dotcom %} ao ambiente do fluxo de trabalho, porque isto também pode afetar o modelo de permissão de {% data variables.product.prodname_dotcom %} concedendo, inadvertidamente, amplo acesso aos colaboradores.

Temos [ um plano no roteiro de {% data variables.product.prodname_dotcom %}](https://github.com/github/roadmap/issues/74) para suportar um fluxo que permite o acesso de todos os repositórios em {% data variables.product.product_name %}, embora ainda não seja um recurso compatível. Atualmente, a única maneira de executar interações privilegiadas entre repositórios é colocar um token de autenticação do {% data variables.product.prodname_dotcom %} ou chave SSH como um segredo dentro do ambiente de fluxo de trabalho. Uma vez que muitos tipos de token de autenticação não permitem acesso granular a recursos específicos, há um risco significativo no uso do tipo incorreto de token, pois ele pode conceder acesso muito mais amplo do que o pretendido.

Esta lista descreve as abordagens recomendadas para acessar os dados do repositório dentro de um fluxo de trabalho, em ordem decrescente de preferência:

1. **O `GITHUB_TOKEN` no ambiente de fluxo de trabalho**
    -  Este token tem um escopo intencional para o único repositório que invocou o fluxo de trabalho, e tem o mesmo nível de acesso que um usuário de acesso de gravação no repositório. O token é criado antes de cada trabalho começar e expira quando o trabalho é finalizado. Para obter mais informações, consulte "[Autenticação com o GITHUB_TOKEN](/actions/configuring-and-managing-workflows/authenticating-with-the-github_token)".
    - O `GITHUB_TOKEN` deve ser usado sempre que possível.
2. **Chave de implantação do repositório**
    - Chaves de implantação são um dos únicos tipos de credenciais que concedem acesso de leitura ou gravação a um único repositório, e podem ser usadas para interagir com outro repositório dentro de um fluxo de trabalho. Para obter mais informações, consulte "[Gerenciar chaves de implantação](/developers/overview/managing-deploy-keys#deploy-keys)".
    - Observe que as chaves de implantação só podem clonar e fazer push para o repositório usando o Git, e não podem ser usada para interagir com a API REST ou o GraphQL. Portanto, elas podem não ser apropriadas para os suas necessidades.
3. **Tokens de {% data variables.product.prodname_github_app %}**
    - {% data variables.product.prodname_github_apps %} podem ser instalados em repositórios selecionados e até mesmo ter permissões granulares nos recursos dentro deles. É possível criar um {% data variables.product.prodname_github_app %} interno na sua organização, instalá-lo nos repositórios os quais você precisa acessar dentro do seu fluxo de trabalho, e autenticar como instalação dentro de seu fluxo de trabalho para acessar esses repositórios.
4. **Tokens de acesso pessoal**
    - Você nunca deve usar tokens de acesso pessoais da sua própria conta. Esses tokens concedem acesso a todos os repositórios nas organizações às quais você tem acesso, bem como a todos os repositórios pessoais na sua conta de usuário. Isto concede indiretamente amplo acesso a todos os usuários com acesso de gravação do repositório no qual se encontra o fluxo de trabalho. Além disso, se você deixar uma organização mais adiante, os fluxos de trabalho que usam este token falharão imediatamente e a depuração deste problema pode ser difícil.
    - Se um token de acesso pessoal for usado, ele deverá ser gerado para uma nova conta que só tenha acesso aos repositórios específicos necessários para o fluxo de trabalho. Observe que esta abordagem não é escalável e deve ser evitada em detrimento de alternativas, como as chaves de implantação.
5. **Chaves SSH em uma conta de usuário**
    - Os fluxos de trabalho nunca devem usar as chaves SSH em uma conta de usuário. Semelhante aos tokens de acesso pessoais, eles concedem permissões de leitura/gravação a todos os seus repositórios pessoais, bem como a todos os repositórios aos quais você tem acesso por meio da associação à organização.  Isto concede indiretamente amplo acesso a todos os usuários com acesso de gravação do repositório no qual se encontra o fluxo de trabalho. Se você pretende usar uma chave SSH porque você só precisa executar clones ou push do repositório, e não precisar interagir com APIs públicas, você deverá usar chaves de implantação individuais.

### Fortalecimento para executores auto-hospedados

Os executores ** hospedados em {% data variables.product.prodname_dotcom %}** executam o código dentro de máquinas virtuais efêmeras e limpas e isoladas. Isso quer isto dizer que não há maneira de comprometer persistentemente este ambiente ou obter, de outra forma, acesso a mais informações do que foram colocadas neste ambiente durante o processo de inicialização.

Executores **auto-hospedados** em {% data variables.product.product_name %} não têm garantias com relação ao funcionamento em máquinas virtuais limpas efêmeras e podem ser persistentemente comprometidos por código não confiável em um fluxo de trabalho.

Como resultado, os executores auto-hospedados quase [nunca devem ser usados para repositórios públicos](/actions/hosting-your-own-runners/about-self-hosted-runners#self-hosted-runner-security-with-public-repositories) em {% data variables.product.product_name %}, porque qualquer usuário pode abrir pull requests contra o repositório e comprometer o ambiente. Da mesma forma, tenha cuidado ao usar executores auto-hospedados em repositórios privados, uma vez que qualquer pessoa que possa bifurcar o repositório e abrir um PR (geralmente aqueles com acesso de leitura ao repositório) são capazes de comprometer o ambiente de executores auto-hospedados, incluindo a obtenção de acesso a segredos e `GITHUB_TOKEN` mais privilegiado, que concede permissões de gravação no repositório.

Você também deve considerar o ambiente das máquinas de executores auto-hospedadas:
- Que informação sensível reside na máquina configurada como um executor auto-hospedado? Por exemplo, chaves SSH privadas, tokens de acesso à API, entre outros.
- A máquina tem acesso à rede a serviços sensíveis? Por exemplo, serviços de metadados do Azure ou AWS. A quantidade de informações confidenciais neste ambiente deve ser limitada ao mínimo, e você deve estar sempre ciente de que qualquer usuário capaz de invocar fluxos de trabalho terá acesso a esse ambiente.
