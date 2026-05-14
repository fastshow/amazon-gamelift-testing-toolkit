FROM amazonlinux:2023
RUN dnf install -y 'dnf-command(config-manager)'
RUN mkdir /local/game
WORKDIR /local/game/
RUN dnf install -y dotnet-sdk-8.0 openssl-libs unzip libicu git
RUN git clone --depth 1 --branch v5.4.0 https://github.com/amazon-gamelift/amazon-gamelift-servers-csharp-server-sdk.git aws-gamelift-sdk-temp
RUN mkdir -p DLL ;\
cd aws-gamelift-sdk-temp/src/GameLiftServerSDK ;\
dotnet publish -c Release -f net8.0 -o /local/game/DLL/ ;\
cd /local/game ;\
rm -rf aws-gamelift-sdk-temp
COPY . /local/game/
RUN dotnet publish SampleGameBuild.csproj -c Release -r linux-x64 --self-contained true -o /local/game/publish ;\
cp /local/game/DLL/*.dll /local/game/publish/ ;\
cp /local/game/log4net.config /local/game/publish/ ;\
cp /local/game/QuizConfig.json /local/game/publish/
CMD tail -f
