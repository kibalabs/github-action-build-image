FROM alpine:3 AS build
ARG MESSAGE=default
RUN echo "$MESSAGE" > /message

FROM alpine:3
COPY --from=build /message /message
RUN echo final >> /message
