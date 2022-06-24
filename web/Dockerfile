# Publish application
FROM registry.access.redhat.com/ubi8/dotnet-60 AS build-env
USER 0
WORKDIR /src
COPY . ./
RUN dotnet restore .
RUN dotnet publish --no-restore -c Release -o /out .

# Build application image
FROM registry.access.redhat.com/ubi8/dotnet-60-runtime
COPY --from=build-env /out /app
ENV ASPNETCORE_URLS=http://*:8080
CMD ["dotnet", "/app/web.dll"]
