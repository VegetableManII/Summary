# 基于Redis分布式锁

```go
type Lock struct {
	Resource  string // 要加锁的资源
	CacheTime int64  // Redis数据缓存时长(毫秒)，也就是自动释放锁的时长

	Key      string // Redis数据项的Key
	Token    string // Redis数据项的Value
	CreateAt int64
}
// 非阻塞式加锁
func (lock *Lock) TryLock() (bool, error) {
	if lock == nil || lock.Key == "" {
		return false, errors.New("this lock is not initialized")
	}

	r := imredis.Master().Cmd("SET", lock.Key, lock.Token, "PX", lock.CacheTime, "NX") // 设置资源的过期时间
	if r.Err != nil {
		return false, r.Err
	}
	if imredis.IsNil(r) {
		return false, nil // 未取得锁
	}
	if result, e := r.Str(); e != nil {
		return false, e
	} else if result != "OK" {
		return false, fmt.Errorf("bad trylock response: %s", r.String())
	}

	return true, nil // 成功取得锁
}
// 阻塞式加锁
func (lock *Lock) Lock() (bool, error) {
	if lock == nil || lock.Key == "" {
		return false, errors.New("this lock is not initialized")
	}

	var pauseTime int64 = 0
	for {
    // 阻塞实现
		time.Sleep(time.Millisecond * time.Duration(pauseTime))
    // 尝试加锁即给资源设置过期时间，加锁成功跳出循环返回
    // 加锁失败
		if ok, e := lock.TryLock(); ok && e == nil {
			return true, nil
		} else if !ok && e == nil {
			if pttl, e2 := imredis.Master().CmdWithReconnectOnIOErr("PTTL", lock.Key).Int64(); e2 != nil {
				return false, e2
			} else if pttl == -2 {
				pauseTime = 0
				continue
			} else if pttl == -1 {
				return false, fmt.Errorf("this lock will never expire. key: %s resource: %s", lock.Key, lock.Resource)
			} else if pttl >= 0 {
				pauseTime = pttl
				continue
			} else {
				return false, fmt.Errorf("unexpected PTTL response. key: %s pttl: %v", lock.Key, pttl)
			}
		} else {
			return false, e
		}
	}
}

```