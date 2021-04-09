# go map 排序

```GO
type PairList []Pair

func (p PairList) Len() int {
	return len(p)
}

func (p PairList) Less(i, j int) bool {
	if p[i].Value < p[j].Value {
		return true
	} else if p[i].Value == p[j].Value {
		if p[i].Key < p[j].Key {
			return false
		} else {
			return true
		}
	} else {
		return false
	}
}

func (p PairList) Swap(i, j int) {
	p[i], p[j] = p[j], p[i]
}

func rank(hashtable map[int]int) PairList {
	pl := make(PairList, len(hashtable))
	i := 0
	for k, v := range hashtable {
		pl[i] = Pair{k, v}
		i++
	}
	//sort.Sort(pl)
	sort.Sort(sort.Reverse(pl))
	return pl
}
```

