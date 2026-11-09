# =============================================================================
# STAGE 1: BUILD STAGE (.NET 8 SDK)
# =============================================================================
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# Copia e restaura dependências em camada isolada para cache otimizado
COPY ["src/ApiService/ApiService.csproj", "ApiService/"]
RUN dotnet restore "ApiService/ApiService.csproj"

# Copia o restante do código e publica em modo Release
COPY src/ .
WORKDIR "/src/ApiService"
RUN dotnet publish "ApiService.csproj" -c Release -o /app/publish /p:UseAppHost=false

# =============================================================================
# STAGE 2: HARDENED RUNTIME STAGE (ALPINE + NON-ROOT USER)
# =============================================================================
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app

# Atualiza pacotes de segurança da imagem base e cria grupo/usuário sem privilégio
RUN apk update && apk upgrade --no-cache && \
    addgroup -g 10001 -S appsecgroup && \
    adduser -u 10001 -S appsecuser -G appsecgroup && \
    chown -R appsecuser:appsecgroup /app

# Variáveis de ambiente restritivas para produção
ENV ASPNETCORE_URLS=http://+:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_EnableDiagnostics=0

COPY --from=build --chown=appsecuser:appsecgroup /app/publish .

# Alterna para usuário não-privilegiado (Mitigação de Container Escape)
USER appsecuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "ApiService.dll"]