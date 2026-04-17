# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o vikunja ./...

# Runtime stage
FROM scratch
COPY --from=builder /app/vikunja /vikunja
USER 1000
ENTRYPOINT ["/vikunja"]