FROM golang:1.13-alpine AS builder
WORKDIR /app
COPY . .
RUN apk add --no-cache git
RUN go build -o go-docker -ldflags="-X 'main.version=${VERSION}'"
# final stage
FROM alpine:latest
WORKDIR /root
RUN apk --no-cache add ca-certificates
RUN apk add curl
COPY --from=builder /app/go-docker .
ENTRYPOINT ./go-docker

EXPOSE 8000
