FROM node:22-bookworm

RUN apt-get update && \
    apt-get install -y curl ca-certificates tar && \
    npm install -g @agentmemory/agentmemory

RUN mkdir -p /root/.agentmemory/bin

RUN curl -fsSL \
https://github.com/iii-hq/iii/releases/download/iii/v0.11.2/iii-aarch64-unknown-linux-gnu.tar.gz \
| tar -xz -C /root/.agentmemory/bin

RUN chmod +x /root/.agentmemory/bin/iii

ENV PATH="/root/.agentmemory/bin:${PATH}"

CMD ["agentmemory"]
