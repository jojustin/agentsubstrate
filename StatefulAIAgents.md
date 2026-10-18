## Increment the counter 
for i in {1..4}; do
  curl -s -X POST \
    -H "ate-target-actor: ate-demo-counter/my-counter-1" \
    http://localhost:8000/
  echo
done

### Verify the actor is running 
kubectl ate get actor my-counter-1 -a ate-demo-counter

## Suspend the actor
kubectl ate suspend actor my-counter-1 -a ate-demo-counter --trace

## Resume the actor
kubectl ate resume actor my-counter-1 -a ate-demo-counter --trace

Send another request -> curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" -i http://localhost:8000/
