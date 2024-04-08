# ssip-trusted-pipeline

## 1. Introdução

Esse repositório contém um conjunto de workflows reusáveis que podem ser usados para criar uma pipeline CI/CD DevSecOps que gera uma imagem Docker, e atestados infalsificáveis para a imagem Docker que podem ser usados por um consumidor para verificar a integridade da imagem, o processo de build, e se a imagem obedece aos requisitos de segurança.

Os atestados assinados estão no formato in-toto, e são gerados por geradores SLSA3, ou pela ferramenta cosign.

A pipeline também faz o deploy da imagem em um cluster k8s usando o arquivo de manifesto fornecido, juntamente com um arquivo kubeconfig que deve ser armazenado como um segredo no repositório do workflow chamador.

## 2. Visão geral do workflow

O [workflow principal](.github/workflows/devsecops-pipeline.yml) desse repositório é acionado por uma chamada de outro workflow, e faz os seguintes jobs:

1. SAST Scan do repositório usando Semgrep
2. Teste unitário do código fonte usando go test
3. Construção da imagem Docker junto com o atestado SLSA, e armazenamento da imagem em um registry temporário.
4. Geração do SBOM da imagem usando a ferramenta trivy
5. Análise de vulnerabilidades da imagem usando a ferramenta trivy
6. Release da imagem no registry do projeto que chamou o workflow
7. Criação e assinatura dos atestados in-toto, usando a ferramenta cosign. Os seguintes atestados são gerados:
   1. Atestado SBOM (cyclonedx)
   2. Atestado de Análise de Vulnerabilidades (vuln)
   3. Atestado de SAST Scan (predicado genérico)
   4. Atestado de Teste Unitário (predicado genérico)
   5. Atestado SLSA (slsaprovenance)
8. Deploy da imagem no cluster k8s usando o arquivo de manifesto fornecido e o arquivo kubeconfig armazenado como um segredo no repositório.

## 2. Como usar

### 2.1. Pré-requisitos

1. Ter um repositório com um projeto em Golang, que contenha arquivos de teste, e um Dockerfile para a construção da imagem Docker.

2. Ter um cluster Kubernetes configurado e acessível. O arquivo de configuração do cluster deve estar configurado como um segredo no repositório.

3. Um arquivo de manifesto de deploy do Kubernetes, que será utilizado para fazer o deploy da imagem Docker no cluster.

4. Um registry de imagens Docker privado que será utilizado para armazenar temporariamente a imagem Docker gerada, antes de ser feito o release da imagem.

### 2.2. Configuração

Em seu repositório, crie um arquivo de workflow que chame [workflow principal](.github/workflows/trusted-pipeline.yml) em um dos jobs.

#### 2.2.1. Inputs

1. `working-directory`: Diretório onde o Dockerfile está localizado. Se omitido o workflow vai buscar o Dockerfile na raiz do repositório.

2. `package-test-path`: Diretório onde os arquivos de teste estão localizados. Se omitido o workflow vai buscar os arquivos de teste na raiz do repositório.

3. `temp-registry`: URL do registry de imagens Docker privado que será utilizado para armazenar temporariamente a imagem Docker gerada, antes de ser feito o release da imagem.

4. `deployment`: Caminho para o manifesto de deployment.

#### 2.2.2. Secrets

1. `intermediary-registry-username`: Usuário do registry de imagens Docker privado que será utilizado para armazenar temporariamente a imagem Docker gerada, antes de ser feito o release da imagem.

2. `intermediary-registry-password`: Senha do registry de imagens Docker privado que será utilizado para armazenar temporariamente a imagem Docker gerada, antes de ser feito o release da imagem.

### 2.3. Exemplo de uso

O repositório [ssip-demo-project](https://github.com/Laerson/ssip-demo-project) contém um exemplo de uso do workflow.

## 3. Verificação dos atestados

O cliente pode verificar a integridade da imagem final usando a ferramenta `cosign verify-attestation`.

Exemplo:

```bash
cosign verify-attestation \
  --certificate-identity="https://github.com/Laerson/ssip-trusted-pipeline/.github/workflows/trusted-pipeline.yml@refs/tags/v1.0.0" \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/laerson/ssip-demo-project:v1.0.0
```

Para mais detalhes sobre a verificação dos atestados, consulte a [documentação do cosign](https://github.com/sigstore/cosign/blob/main/doc/cosign_verify-attestation.md).

## 4. Outro

Para detalhes sobre o modelo de segurança, consulte a [especificação técnica do SLSA](https://github.com/slsa-framework/slsa-github-generator/blob/main/SPECIFICATIONS.md)

Esse repositório foi criado como parte do projeto de pesquisa "SSIP: Software Supply Chain Security" do Laboratório de Sistemas Distribuídos da UFCG.
