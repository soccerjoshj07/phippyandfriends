# Build
FROM golang:1.13.4 as builder

WORKDIR /build
COPY main.go .
RUN GO111MODULE=on; go mod init captainkube && go get k8s.io/client-go@v0.17.2 && go get -d -v
# temporary workaroud for https://github.com/Azure/phippyandfriends/issues/30
#RUN cd $GOPATH/src/k8s.io/klog && git checkout v0.4.0 && cd -
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# Run
FROM alpine:latest  

RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /build/app .
CMD ["./app"]
