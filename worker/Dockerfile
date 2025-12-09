FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /source
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore
FROM mcr.microsoft.com/dotnet/runtime:7.0
WORKDIR /app
COPY --from=build /app/publish .
# utilisateur non root
RUN useradd -m appuser
COPY --from=build /app/publish .
RUN chown -R appuser /app
USER appuser

CMD ["dotnet", "Worker.dll"]